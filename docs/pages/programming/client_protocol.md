---
title: Comdb2 Client Protocol
keywords: code
sidebar: mydoc_sidebar
permalink: client_protocol.html
---

## Comdb2 Client Protocol

Comdb2 client protocol is used between Comdb2 client and server. To start a connection the client needs to know server
port which is managed by pmux. The queries and response is sent using google protobufs
[https://developers.google.com/protocol-buffers/](https://developers.google.com/protocol-buffers/).

Getting database port number from pmux
---------------------------------------
Each comdb2 database uses pmux to assign port to it, and this port number is
persistent for multiple database runs. Clients ask pmux which port to connect to
for specific database. The default pmux port is 5105. 

To get port number of db with dbname as `<dbname>`, portmux accepts newline terminated string 
in the format:
```
"get comdb2/replication/<dbname>\n"
```
and gives back port number in 32 byte character array as reply.


Example code in Python to get database port:

```python
def portmux_get(host, dbname):
  client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  service = "comdb2/replication"
  command = "get" + service + dbname + "\n"
  client_socket.connect(host, 5105)
  client_socket.send(command)
  byts = client_socket.recv(32)
  port = int(byts)
  return port
```  
  

Starting the database connection
--------------------------------

Comdb2 server expects the first line of every new connection from client to be newline terminated "newsql". Once client
sends this a new appsock thread is started at the server side, which waits for queries from the client. To view
active appsock connections on server one can run stored procedure "exec procedure sys.cmd.send('stat thr')" using cdb2sql.
This needs to be sent only once at the start of new connection.

Example code in  python:

```python
  port = portmux_get(comdb2dbhost, comdb2dbname)
  client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  client_socket.connect(comdb2dbhost, int(port)))
  client_socket.send("newsql\n")
```


SQL Header
------
The header is 16 byte message that is sent before every request/response after the connection is established.
The first 4 bytes contains information about type of request/response and the last 4 bytes have info about size. 
8 bytes in the middle are not used yet.  The values in header are stored in network byte order (big endian).

example code in python:

```python
  byts = struct.pack("!iiii",sqlquery_pb2.CDB2QUERY,0,0,size)
  client_socket.send(byts)
  client_socket.send(send_data)
```  

Executing query on server:
-------------------------

### SQL Query Protobuf

```
message CDB2_FLAG {
  required int32 option = 1;
  required int32 value = 2;
}

enum CDB2ClientFeatures {
    SKIP_ROWS            = 1;
    ALLOW_MASTER_EXEC    = 2;
    ALLOW_MASTER_DBINFO  = 3;
    ALLOW_QUEUING  = 4;
}

message CDB2_SQLQUERY {
  required string dbname = 1;
  required string sql_query = 2;
  repeated CDB2_FLAG flag = 3;
  required bool little_endian = 4;
  message bindvalue {
    required string varname = 1;
    required int32  type    = 2;
    required bytes  value   = 3;
    optional bool   isnull  = 4 [default = false];
    optional int32  index   = 5;
  }
  repeated bindvalue bindvars = 5;
  optional string tzname = 6;
  repeated string set_flags = 7;
  repeated int32 types = 8;
  optional string mach_class = 9 [default = "unknown"];
  optional bytes cnonce = 10;
  message snapshotinfo {
    required int32  file    = 1;
    required int32  offset  = 2;
  }
  optional snapshotinfo snapshot_info = 11; 
  optional int64 skip_rows = 12; // number of rows to be skipped, -1 (skip all rows)
  optional int32 retry = 13  [default = 0];
  // if begin retry < query retry then skip all the rows from server, if same then skip (skip_rows)
  repeated int32 features = 14; // Client can negotiate on this.
}


message CDB2_DBINFO {
  required string dbname = 1;
  required bool little_endian = 2;
  optional bool want_effects = 3;
}

message CDB2_QUERY {
  optional CDB2_SQLQUERY sqlquery = 1;
  optional CDB2_DBINFO   dbinfo = 2;
}
```

|Name|Description
|----|----------------------------------------------------------
|dbname| This is the database name to which query is being sent
|sql_query| The query to be executed
| flag| The client options and their values to be sent to server
|little_endian| This flag tells if the response column values should come in little endian or big endian format.
| bindvars| This contains the name, the type and pointer to the value to be bound, a faster option of binding by index is also available.
|tzname| Timezone of datetime fields in result set
|set_flags | This contains all set commands e.g. "set transaction read committed", "set timezone GMT" etc. This needs to be attached to first query and  persist for the database connection.
| types| The data type of result set, if present it should have data type of all the resulting columns.
| mach_class| this option tells to which machine class does the client belongs.
| cnonce | Client nonce, this is to prevent replays in high availability transactions
| snapshot_info| This is required when retry is done in HA transaction, the information comes from server at the start of HA transaction.
| skip_rows| number of rows to be skipped in result set, -1 (skip all rows)
| retry| tells if this query is retry by client (the cnonce number should be same), if begin retry < query retry then skip all the rows from server, if same then skip (skip_rows)
| features| not used



Sending dbinfo request and getting database cluster information
------------------------------------

### Dbinfo Response Protobuf

```
message CDB2_DBINFORESPONSE {
    message nodeinfo {
        required string  name = 1;
        required int32   number = 2;
        required int32   incoherent = 3;
        optional int32   room = 4;
        optional int32   port = 5;
    }
    required nodeinfo master = 1;
    repeated nodeinfo nodes  = 2;     // These are replicant nodes, we need to parse through these only.
}

```
After connecting to database to get db cluster info the client has to send dbinfo request to server. For this 
the client has to use dbinfo object in CDB2_QUERY protobuf object.  This object requires two required
fields dbname and little_endian. The SQL header is sent to server with the type and size of the request and
then dbinfo request is sent. 

The server gives dbinfo reply along with the header. SQL header is read from server and the
header type in this case is DBINFO_RESPONSE (1005). The response from database contains information about the master
nodes and all the nodes (including master). The information that it contains about each node is in the table below.


|Name|Description
|----|----------------------------------------------------------
|name | Hostname of the db node
|port | Port number of the db node
|number | node number, or host id of the connected node (this is internal to Bloomberg)
|incoherent | If this flag is set, then it means that the node is incoherent
|room | Datacenter id of the node


Example in python:

```python
query = sqlquery_pb2.CDB2_QUERY()
query.dbinfo.dbname = dbname
query.dbinfo.little_endian  = 0


size = query.ByteSize();
send_data = query.SerializeToString();

byts = struct.pack("!iiii",sqlquery_pb2.CDB2QUERY,0,0,size)
client_socket.send(byts) #send newsql header
client_socket.send(send_data)

response = sqlresponse_pb2.CDB2_DBINFORESPONSE()
```

Sending SQL query request to server:
----------------------------

After the connection is established with server, the client can fill the query information in CDB2_QUERY protobuf
object. To send the sql query, the client should fill in the info in sqlquery object. Three required fields of this 
object are sql_query, dbname and little_endian (all explained above). The client has to make this object and then
send a sql header to server with type as CDB2QUERY and size as object size, followed by the CDB2_QUERY protobuf object.


Example SQL Query in python:
------------------------

```python
query = sqlquery_pb2.CDB2_QUERY()
query.sqlquery.sql_query = "select 1";
query.sqlquery.dbname = "mohitdb1"
query.sqlquery.little_endian  = 0
query.sqlquery.tzname  = "America/New_York"

size = query.ByteSize();
send_data = query.SerializeToString();

byts = struct.pack("!iiii",sqlquery_pb2.CDB2QUERY,0,0,size)
client_socket.send(byts)
client_socket.send(send_data)
```

SQL Query Response
------------------


### Sql response Protobuf


```
enum CDB2_ColumnType {
    INTEGER      = 1;
    REAL         = 2;
    CSTRING      = 3;
    BLOB         = 4;
    DATETIME     = 6;
    INTERVALYM   = 7;
    INTERVALDS   = 8;
    DATETIMEUS   = 9;
    INTERVALDSUS = 10;
}

enum CDB2_ErrorCodes {
    OK                 =  0;
    DUP_OLD            =  1;
    CONNECT_ERROR      = -1;
    NOTCONNECTED       = -2;

    PREPARE_ERROR      = -3;
    PREPARE_ERROR_OLD  = 1003;

    IO_ERROR           = -4;
    INTERNAL           = -5;
    NOSTATEMENT        = -6;
    BADCOLUMN          = -7;
    BADSTATE           = -8;
    ASYNCERR           = -9;
    OK_ASYNC              = -10;

    INVALID_ID              = -12;
    RECORD_OUT_OF_RANGE     = -13;

    REJECTED                = -15;
    STOPPED                 = -16;
    BADREQ                  = -17;
    DBCREATE_FAILED         = -18;

    THREADPOOL_INTERNAL     = -20;  /* some error in threadpool code */
    READONLY                = -21;

    NOMASTER                = -101;
    UNTAGGED_DATABASE       = -102;
    CONSTRAINTS             = -103;
    DEADLOCK                =  203;

    TRAN_IO_ERROR           = -105;
    ACCESS                  = -106;

    TRAN_MODE_UNSUPPORTED   = -107;

    VERIFY_ERROR            = 2;
    FKEY_VIOLATION          = 3;
    NULL_CONSTRAINT         = 4;

    CONV_FAIL               = 113;
    NONKLESS                = 114;
    MALLOC                  = 115;
    NOTSUPPORTED            = 116;

    DUPLICATE               =  299;
    TZNAME_FAIL             =  401;
    UNKNOWN                 =  300;
}

enum ResponseHeader {
   SQL_RESPONSE    = 1002;
   DBINFO_RESPONSE = 1005;
   SQL_EFFECTS     = 1006;
   SQL_RESPONSE_PING = 1007; // SQL_RESPONSE + requesting ack
   SQL_RESPONSE_PONG = 1008;
}

enum ResponseType {
  COLUMN_NAMES  = 1;
  COLUMN_VALUES = 2;
  LAST_ROW      = 3;
  COMDB2_INFO   = 4; // For info about features, or snapshot file/offset etc
}

enum CDB2ServerFeatures {
    SKIP_ROWS    = 1;
}

message CDB2_DBINFORESPONSE {
    message nodeinfo {
        required string  name = 1;
        required int32   number = 2;
        required int32   incoherent = 3;
        optional int32   room = 4;
        optional int32   port = 5;
    }
    required nodeinfo master = 1;
    repeated nodeinfo nodes  = 2;     // These are replicant nodes, we need to parse through these only.
}

message CDB2_EFFECTS {
    required int32 num_affected = 1;
    required int32 num_selected = 2;
    required int32 num_updated  = 3;
    required int32 num_deleted  = 4;
    required int32 num_inserted = 5;
}

message CDB2_SQLRESPONSE {
    required ResponseType response_type=1; // enum ReponseType
    message column {
        optional CDB2_ColumnType type = 1;
        required bytes value = 2;
        optional bool isnull = 3 [default = false];  
    }
    repeated column value=2;
    optional CDB2_DBINFORESPONSE dbinforesponse =3;
    required CDB2_ErrorCodes error_code = 4;
    optional string error_string = 5;
    optional CDB2_EFFECTS effects = 6; 
    message snapshotinfo {
        required int32 file = 1;
        required int32 offset = 2;
    }
    optional snapshotinfo snapshot_info=7;
    // in case of retry, this will be used to identify the rows which need to be discarded
    optional uint64 row_id   = 8; 
    repeated CDB2ServerFeatures  features = 9; // This can tell client about features enabled in comdb2
}
```

|Name|Description
|----|------------------
|response_type | Tells if the message contains information about names, values or the cluster nodes.
| value| Contains the column names in the case when response_type is COLUMN_NAMES and the values if its COLUMN_VALUES.  The endianess of the values is decided by the little_endian flag given in query.
| dbinforesponse| The information about the comdb2 cluster.
| error_code| The error code in the case of failure of the query.
| error_string| The error string in the case of failure of the query.
| effects| This has the information of rows inserted/deleted/updated/selected for the transaction.
| snapshot_info| The snapshot info sent by server, this is required when retry is done in HA transaction
| row_id|  in case of retry, this is used to identify the rows which need to be discarded (skip_rows in query)
|features | This can tell client about features supported in comdb2


The first response from server contains information about column names. The response type for the first response is COLUMN_NAMES.

Example in python:

```python
firstresponse = sqlresponse_pb2.CDB2_SQLRESPONSE()
byts = client_socket.recv(16) #read newsql header
if (struct.unpack("!iiii",byts)[0] != sqlresponse_pb2.SQL_RESPONSE):
    sys.stdout.write("Invalid reply from server.\n")
    return
size = (struct.unpack("!iiii",byts))[3]
firstresponse.ParseFromString(client_socket.recv(size))
if (firstresponse.response_type != sqlresponse_pb2.COLUMN_NAMES):
    sys.stdout.write("Invalid reply from server.\n")
    return
if (firstresponse.error_code != sqlresponse_pb2.OK):
    sys.stdout.write("Error: rc="+str(firstresponse.error_code)+ 
    " str="+firstresponse.error_string+"\n")
    return
```

The server response after the first response contains the column values. The response type for this response is COLUMN_VALUES.
This happens until the last row, in which case the response type is LAST_ROW. If the table is empty then the response that comes
after COLUMN_NAMES is LAST_ROW.

Example in python:

```python
#get first column value
response = sqlresponse_pb2.CDB2_SQLRESPONSE()
byts = client_socket.recv(16) #read newsql header
if (struct.unpack("!iiii",byts)[0] != sqlresponse_pb2.SQL_RESPONSE):
    sys.exit("Invalid reply from server.")
size = (struct.unpack("!iiii",byts))[3] 
response.ParseFromString(client_socket.recv(size))

while response.response_type == sqlresponse_pb2.COLUMN_VALUES:
if (response.error_code != sqlresponse_pb2.OK):
  sys.stdout.write("Error: rc="+str(response.error_code)+ " str="+response.error_string+"\n")
  return
... do the work ...
size = 0     
while (size == 0):
  byts = client_socket.recv(16) #read newsql header
  size = (struct.unpack("!iiii",byts))[3] #newsql header with no data is server heartbeat
response.ParseFromString(client_socket.recv(size))
```

Example comdb2 python client which prints the result of query:
------------------

```python
import socket
import struct
import sys
import sqlquery_pb2
import sqlresponse_pb2

def portmux_get(host, dbname):
  client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  service = "comdb2/replication/"
  command = "get " + service + dbname  +"\n"
  client_socket.connect((host, 5105))
  client_socket.send(command)
  byts = client_socket.recv(32)
  port = int(byts)
  return port
  

if len(sys.argv) != 4:
    print "Run with the following arguments"
    print sys.argv[0] + " <dbname> <machine> <query>"
    sys.exit()
    

dbname  = sys.argv[1]
machine = sys.argv[2]
client_query   = sys.argv[3]


port =  portmux_get(machine, dbname)
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.connect((machine, int(port)))
client_socket.send("newsql\n")


query = sqlquery_pb2.CDB2_QUERY()
query.sqlquery.sql_query = client_query;
query.sqlquery.dbname = dbname
query.sqlquery.little_endian  = 0
query.sqlquery.tzname = "America/New_York"

size = query.ByteSize();
send_data = query.SerializeToString();

byts = struct.pack("!iiii",sqlquery_pb2.CDB2QUERY,0,0,size)
client_socket.send(byts) #send newsql header
client_socket.send(send_data)

firstresponse = sqlresponse_pb2.CDB2_SQLRESPONSE()
byts = client_socket.recv(16) #read newsql header
if (struct.unpack("!iiii",byts)[0] != sqlresponse_pb2.SQL_RESPONSE):
    sys.stdout.write("Invalid reply from server.\n")
    sys.exit()
size = (struct.unpack("!iiii",byts))[3]
firstresponse.ParseFromString(client_socket.recv(size))
if (firstresponse.response_type != sqlresponse_pb2.COLUMN_NAMES):
    sys.stdout.write("Invalid reply from server.\n")
    sys.exit()
if (firstresponse.error_code != sqlresponse_pb2.OK):
    sys.stdout.write("Error: rc="+str(firstresponse.error_code)+ 
    " str="+firstresponse.error_string+"\n")
    sys.exit()
#get first column value
response = sqlresponse_pb2.CDB2_SQLRESPONSE()
byts = client_socket.recv(16) #read newsql header
if (struct.unpack("!iiii",byts)[0] != sqlresponse_pb2.SQL_RESPONSE):
    sys.exit("Invalid reply from server.")
size = (struct.unpack("!iiii",byts))[3] 
response.ParseFromString(client_socket.recv(size))

while response.response_type == sqlresponse_pb2.COLUMN_VALUES:
  if (response.error_code != sqlresponse_pb2.OK):
    sys.stdout.write("Error: RC="+str(response.error_code)+ 
    " str="+response.error_string+"\n")
    sys.exit()
  i = 0
  sys.stdout.write("(")
  for column in response.value:
       if (i > 0) : 
         sys.stdout.write(", ")
       sys.stdout.write(firstresponse.value[i].value)
       sys.stdout.write("=")
       if (column.isnull== True):
         sys.stdout.write("NULL")
       elif (firstresponse.value[i].type == sqlresponse_pb2.INTEGER):
         vallong = struct.unpack("!q", column.value)[0]
         sys.stdout.write(str(vallong))
       elif (firstresponse.value[i].type == sqlresponse_pb2.REAL):
         valreal = struct.unpack("!d", column.value)[0]
         sys.stdout.write(str(valreal))
       elif (firstresponse.value[i].type == sqlresponse_pb2.DATETIME):
         datetime = struct.unpack("!iiiiiiiiiI36s", column.value)
         sys.stdout.write("\""+str(datetime[5]+1900)+"-"+str(datetime[4]+1)+"-"+
                          str(datetime[3])+"T"+str(datetime[2]).zfill(2)+
                          str(datetime[1]).zfill(2)+str(datetime[0]).zfill(2)+"."+
                          str(datetime[9]).zfill(3)+ " " +datetime[10]+"\"")
       elif (firstresponse.value[i].type == sqlresponse_pb2.DATETIMEUS):
         datetime = struct.unpack("!iiiiiiiiiI36s", column.value)
         sys.stdout.write("\""+str(datetime[5]+1900)+"-"+str(datetime[4]+1)+"-"+
                          str(datetime[3])+"T"+str(datetime[2]).zfill(2)+
                          str(datetime[1]).zfill(2)+str(datetime[0]).zfill(2)+"."+
                          str(datetime[9]).zfill(6)+
                          " " +datetime[10]+"\"")
       elif (firstresponse.value[i].type == sqlresponse_pb2.INTERVALDS):
         intervalds = struct.unpack("!iIIIII", column.value)
         sign = ""
         if (intervalds[0] < 0):
             sign = "-"
         sys.stdout.write("\""+sign+ str(intervalds[1]) + " " +
                          str(intervalds[2])+":"+str(intervalds[3])+":"+
                          str(intervalds[4])+"."+str(intervalds[5]).zfill(3)+"\"")
       elif (firstresponse.value[i].type == sqlresponse_pb2.INTERVALDSUS):
         intervalds = struct.unpack("!iIIIII", column.value)
         sign = ""
         if (intervalds[0] < 0):
             sign = "-"
         sys.stdout.write("\""+sign+ str(intervalds[1]) + " " +
                          str(intervalds[2])+":"+str(intervalds[3])+":"+
                          str(intervalds[4])+"."+str(intervalds[5]).zfill(6)+"\"")         
       elif (firstresponse.value[i].type == sqlresponse_pb2.BLOB):
         sys.stdout.write("x'")
         sys.stdout.write(column.value.encode("hex"))
         sys.stdout.write("'")
       else:    
         sys.stdout.write("'")
         sys.stdout.write(column.value)
         sys.stdout.write("'")
       i = i + 1
  size = 0     
  while (size == 0):
    byts = client_socket.recv(16) #read newsql header
    size = (struct.unpack("!iiii",byts))[3] #newsql header with no data is server heartbeat
  response.ParseFromString(client_socket.recv(size))
  sys.stdout.write(")\n")  
```

Resetting the connection:
----------------------------
If your connection goes in an unknown transactional state, you can reuse the connection by sending 
CDB2RequestType RESET in SQL header. This will reset the client state on server.

Example in python:

```python
  byts = struct.pack("!iiii",sqlquery_pb2.RESET,0,0,0)
  client_socket.send(byts)
  client_socket.send(send_data)
```
