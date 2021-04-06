
## ProtoBuf and gRPC
### protobuf
Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. 
 
To install protobuf, you need to install 
- the protocol compiler: i.e. protoc, used to compile .proto files (define the structure for the data you want to serialize) to specific language, for example, .pb.go (includes data access classes for messages in proto file, such as methods that serialize/parse the whole structure to/from raw bytes) in Golang
- the protobuf runtime: provides marshaling and unmarshaling between a protobuf message and the binary wire format in bytes
 
line ```option go_package = "tutorial";``` in .proto will generate line ```package tutorial``` in .pb.go. 

each field in the message definition has a unique number. These numbers are used to identify your fields in the message binary format, and should not be changed once your message type is in use. Note that field numbers in the range 1 through 15 take one byte to encode, including the field number and the field's type. Field numbers in the range 16 through 2047 take two bytes. So you should reserve the field numbers 1 through 15 for very frequently occurring message elements. Remember to leave some room for frequently occurring elements that might be added in the future.
 
Required Is Forever. You should be very careful about marking fields as required. If at some point you wish to stop writing or sending a required field, it will be problematic to change the field to an optional field – old readers will consider messages without this field to be incomplete and may reject or drop them unintentionally.
 
When a message is parsed, if it does not contain an optional element, the corresponding field in the parsed object is set to the default value for that field. The default value can be specified as part of the message description. If the default value is not specified for an optional element, a type-specific default value is used instead.
 
```proto
message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;

  google.protobuf.Timestamp last_updated = 5;
}

// Our address book file is just one of these.
message AddressBook {
  repeated Person people = 1;
}
```
 
If you update a message type by entirely removing a field, or commenting it out, future users can reuse the field number when making their own updates to the type. This can cause severe issues if they later load old versions of the same .proto, including data corruption, privacy bugs, and so on. One way to make sure this doesn't happen is to specify that the field numbers (and/or names, which can also cause issues for JSON serialization) of your deleted fields are reserved. The protocol buffer compiler will complain if any future users try to use these field identifiers.
 
What if the message type you want to use as a field type is already defined in another .proto file? You can use definitions from other .proto files by importing them. 
 
Extensions let you declare that a range of field numbers in a message are available for third-party extensions. An extension is a placeholder for a field whose type is not defined by the original .proto file. This allows other .proto files to add to your message definition by defining the types of some or all of the fields with those field numbers.
 
```proto
message Foo {
  // ...
  // range of field numbers [100, 199] in Foo is reserved for extensions
  extensions 100 to 199;

}
extend Foo {
    // add new fields to Foo in their own .proto files that import your .proto, 
    // using field numbers within your specified range
  optional int32 bar = 126;
}
```
### gRPC
Remote procedure call packages all have a simple goal: to make the
process of executing code on a remote machine as simple and straightforward as calling a local function.

protobuf in gRPC
- Interface Definition Language (IDL) for describing both the service interface and the structure of the payload messages 
- underlying message interchange format of gRPC 

```proto
// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  // Sends another greeting
  rpc SayHelloAgain (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. gRPC uses protoc with two plugins in ```$(go env GOPATH)/bin``` (Update your PATH so that the protoc compiler can find the plugins) to generate code from your proto file: 

```shell
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    helloworld/helloworld.proto
```
 
- regular protocol buffer code for populating, serializing, and retrieving your message types, for example, helloworld.pb.go, by calling plugin protoc-gen-go indirectly through protoc 

- generated gRPC client and server code, for example, helloworld_grpc.pb.go, by calling plugin protoc-gen-go-grpc indirectly through protoc
  
  - On the server side, the server use hand-written codes to implements this generated interface and runs a gRPC server to handle client calls. The gRPC infrastructure listen and decodes incoming requests, dispatch them to the right service implementation, executes service methods, and encodes service responses.
  - On the client side, the client has a local object known as stub (for some languages, the preferred term is client) that implements the same methods as the service. The client use hand-written codes to call those generated methods on the local object (stub), wrapping the parameters for the call in the appropriate protocol buffer message type - gRPC looks after sending the request(s) to the server and returning the server’s protocol buffer response(s).

The following codes are excerpt from [gRPC quick start](https://www.grpc.io/docs/languages/go/quickstart/) and [hello world gRPC](https://github.com/grpc/grpc-go/tree/master/examples/helloworld)
```go
// in func main() of helloworld/greeter_client/main.go 
// this is hand written code 

// To call service methods, we first need to create a gRPC channel to communicate with the server. 
// We create this by passing the server address and port number to grpc.Dial() as address:
conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
// Once the gRPC channel is setup, we need a client stub to perform RPCs. 
// We get it using the NewRouteGuideClient method provided by the pb package generated from the example .proto file
c := pb.NewGreeterClient(conn)

// Calling the simple RPC is nearly as straightforward as calling a local method. 
// we call the method on the stub we got earlier. 
// In our method parameters we create and populate a request protocol buffer object. 
// We also pass a context.Context object which lets us change our RPC’s behavior if necessary, such as time-out/cancel an RPC in flight. 
r, err = c.SayHelloAgain(ctx, &pb.HelloRequest{Name: name})

// in helloworld/helloworld/helloworld_grpc.pb.go
// this is auto generated code

func (c *greeterClient) SayHelloAgain(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHelloAgain", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

```go
// in helloworld/greeter_server/main.go
// this is hand written code

// implement interface, the actual worker
func (s *server) SayHelloAgain(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello again " + in.GetName()}, nil
}

func main() {
  // Specify the port we want to use to listen for client requests 
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
  // Create an instance of the gRPC server 
	s := grpc.NewServer()
  // Register our service implementation with the gRPC server
	pb.RegisterGreeterServer(s, &server{})
  // Call Serve() on the server with our port details
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```


four kinds of service method:
- Unary RPCs where the client sends a single request to the server and gets a single response back, just like a normal function call.
- Server streaming RPCs where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. 
- Client streaming RPCs where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them and return its response. 
- Bidirectional streaming RPCs where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. 

More complete example will be at [basic guide](https://www.grpc.io/docs/languages/go/basics/) and code is in [route guide](https://github.com/grpc/grpc-go/tree/master/examples/route_guide). Four different kinds of service method will lead to four different hand-written codes.

RPC life cycle of Unary RPC
- Once the client calls a stub method, the server is notified that the RPC has been invoked with the client’s metadata for this call, the method name, and the specified deadline if applicable.
- The server can then either send back its own initial metadata (which must be sent before any response) straight away, or wait for the client’s request message. Which happens first, is application-specific.
- Once the server has the client’s request message, it does whatever work is necessary to create and populate a response. The response is then returned (if successful) to the client together with status details (status code and optional status message) and optional trailing metadata.
- If the response status is OK, then the client gets the response, which completes the call on the client side.

