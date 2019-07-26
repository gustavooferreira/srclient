Golang Client for Confluent's Schema Registry
=====================================================

<img align="left" width="150" height="150" src="images/Gopher_Apache_Kafka_v1.png">

**srclient** is a Golang client for [Confluent Schema Registry](https://www.confluent.io/confluent-schema-registry/), a software that provides a RESTful interface for developers to define standard schemas for their events, share them across the organization and safely evolve them in a way that is backward compatible and future proof. Using this client allows developers to build Golang applications that need to write and read records to/from [Apache Kafka](https://kafka.apache.org/) can use Schema Registry as their single-source-of-truth for schemas, while still allowing producers and consumers to be decoupled from each other. Producers interact with Schema Registry to fetch schemas and use it to serialize records, and consumers can use it as well to fetch the same schema and use it to deserialize records. Moreover, Schema Registry provides schema enforcement when schemas are based on [Avro](https://avro.apache.org/). You can read more about the benefits of using Schema Registry [here](https://www.confluent.io/blog/schemas-contracts-compatibility).

Features:

- **Simple to Use** - This client provides a very high-level abstraction over the operations that developers writing applications for Apache Kafka typically need. Thus, it will feel natural for them using the functions that this client provides. Moreover, developers don't need to handle low-level HTTP details to communicate with Schema Registry.

- **Performance** - This client provides caching capabilities. This means that any data retrieved from Schema Registry can be cached locally to improve the performance of subsequent requests. This allows applications that are not co-located with Schema Registry to reduce the latency necessary on each request. This functionality can be disabled programmatically.

- **Confluent Cloud** - Go developers using [Confluent Cloud](https://www.confluent.io/confluent-cloud/) can use this client to interact with the fully managed Schema Registry, which provides important features like schema enforcement that enable teams to reduce deployment issues by governing the schema changes as they evolve.

This client creates codec's based on the Avro support from the [goavro](https://github.com/linkedin/goavro) project. Developers can use these codec's to encode and decode from both binary and textual JSON Avro data. If you are using generated code based on Avro compilers, you can disable the codec creation programmatically.

**License**: [Apache License v2.0](http://www.apache.org/licenses/LICENSE-2.0)

Installing
-------------------

Manual install:
```bash
go get -u github.com/riferrei/srclient
```

Golang import:
```golang
import "github.com/riferrei/srclient"
```

Examples
-------------------

**Producer**

```golang
import (
	"encoding/binary"
	"encoding/json"
	"fmt"
	"io/ioutil"

	"github.com/google/uuid"
	"github.com/riferrei/srclient"
	"gopkg.in/confluentinc/confluent-kafka-go.v1/kafka"
)

type ComplexType struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func main() {

	topic := "myTopic"

	// 1) Create the producer as you would normally do using Confluent's Go client
	p, err := kafka.NewProducer(&kafka.ConfigMap{"bootstrap.servers": "localhost"})
	if err != nil {
		panic(err)
	}
	defer p.Close()

	go func() {
		for event := range producer.Events() {
			switch ev := event.(type) {
			case *kafka.Message:
				message := ev
				if ev.TopicPartition.Error != nil {
					fmt.Printf("Error delivering the message '%s'\n", message.Key)
				} else {
					fmt.Printf("Message '%s' delivered successfully!\n", message.Key)
				}
			}
		}
	}()

	// 2) Fetch the latest version of the schema, or create a new one if it is the first
	schemaRegistryClient := srclient.CreateSchemaRegistryClient("http://localhost:8081")
	schema, err := schemaRegistryClient.GetLatestSchema(topic, false)
	if schema == nil {
		schemaBytes, _ := ioutil.ReadFile("complexType.avsc")
		schema, err = schemaRegistryClient.CreateSchema(topic, string(schemaBytes), false)
		if err != nil {
			panic(fmt.Sprintf("Error creating the schema %s", err))
		}
	}
	schemaIDBytes := make([]byte, 4)
	binary.BigEndian.PutUint32(schemaIDBytes, uint32(schema.ID))

	// 3) Serialize the record using the schema provided by the client,
	// making sure to include the schema id as part of the record.
	newComplexType := ComplexType{ID: 1, Name: "Gopher"}
	value, _ := json.Marshal(newComplexType)
	native, _, _ := schema.Codec.NativeFromTextual(value)
	valueBytes, _ := schema.Codec.BinaryFromNative(nil, native)

	var recordValue []byte
	recordValue = append(recordValue, byte(0))
	recordValue = append(recordValue, schemaIDBytes...)
	recordValue = append(recordValue, valueBytes...)

	key, _ := uuid.NewUUID()
	p.Produce(&kafka.Message{
		TopicPartition: kafka.TopicPartition{
			Topic: &topic, Partition: kafka.PartitionAny},
		Key: []byte(key.String()), Value: recordValue}, nil)

	p.Flush(15 * 1000)

}
```

**Consumer**

```golang
import (
	"encoding/binary"
	"fmt"

	"github.com/riferrei/srclient"
	"gopkg.in/confluentinc/confluent-kafka-go.v1/kafka"
)

func main() {

	// 1) Create the consumer as you would
	// normally do using Confluent's Go client
	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost",
		"group.id":          "myGroup",
		"auto.offset.reset": "earliest",
	})
	if err != nil {
		panic(err)
	}
	c.SubscribeTopics([]string{"myTopic", "^aRegex.*[Tt]opic"}, nil)

	// 2) Create a instance of the client to retrieve the schemas for each message
	schemaRegistryClient := srclient.CreateSchemaRegistryClient("http://localhost:8081")

	for {
		msg, err := c.ReadMessage(-1)
		if err == nil {
			schemaID := binary.BigEndian.Uint32(msg.Value[1:5])
			// 3) Recover the schema id from the message and use the
			// client to retrieve the schema from Schema Registry.
			// Then use it to deserialize the record accordingly.
			schema, err := schemaRegistryClient.GetSchema(int(schemaID))
			if err != nil {
				panic(fmt.Sprintf("Error getting the schema with id '%d' %s", schemaID, err))
			}
			native, _, _ := schema.Codec.NativeFromBinary(msg.Value[5:])
			value, _ := schema.Codec.TextualFromNative(nil, native)
			fmt.Printf("Here is the message %s\n", string(value))
		} else {
			fmt.Printf("Error consuming the message: %v (%v)\n", err, msg)
		}
	}

	c.Close()
	
}
```

Both examples have been created using [Confluent's Golang for Apache Kafka<sup>TM</sup>](https://github.com/confluentinc/confluent-kafka-go).

Contributing
------------
Contributions to the code, examples, documentation, et.al, are very much appreciated.