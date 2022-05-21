# grpc_otel

Here you can see my practice for open telemetry in golang . 
this is an example of nested span between server and client 

```go
package cmd

import (
	"context"
	"fmt"
	"os"
	"time"

	"github.com/spf13/cobra"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/jaeger"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.7.0"
	"go.uber.org/zap"
	"google.golang.org/grpc/metadata"

	kangaroo "gitlab.snapp.ir/drx/sample/internal/grpc/proto"
	"gitlab.snapp.ir/drx/sample/internal/log"
	client "gitlab.snapp.ir/drx/sample/pkg/client"
)

var grpcClientCmd = &cobra.Command{
	Use: "client",
	Run: runClient,
}

func runClient(_ *cobra.Command, _ []string) {

	exp, err := jaeger.New(jaeger.WithAgentEndpoint(
		jaeger.WithAgentHost("localhost"),
		jaeger.WithAgentPort("6831")))
	if err != nil {
		log.Logger.Fatal("error in setting jaeger endpoint", zap.Error(err))
	}

	hostname, err := os.Hostname()
	if err != nil {
		log.Logger.Fatal("error in getting hostname", zap.Error(err))
	}

	traceProvider := sdktrace.NewTracerProvider(
		// Always be sure to batch in production.
		sdktrace.WithBatcher(exp),
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
		// Record information about this application in a resource.
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceInstanceIDKey.String(hostname),
			semconv.ServiceNameKey.String("client"),
			semconv.ServiceVersionKey.String("0.01"),
		)),
	)

	deferFunc := func() {
		// Do not make the application hang when it is shutdown.
		ctx, cancel := context.WithTimeout(context.Background(), time.Second*1)
		defer cancel()

		err := traceProvider.Shutdown(ctx)
		if err != nil {
			log.Logger.Fatal("error in shutting down telemetry", zap.Error(err))
		}
	}

	otel.SetTracerProvider(traceProvider)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))

	fmt.Print("setup otel")
	defer deferFunc()

	c, conn := client.GetClient("localhost:50052")
	defer conn.Close()

	// ================================================
	ctxx := metadata.NewOutgoingContext(context.Background(), metadata.Pairs(
		"user", "someone",
	))
	ctx, cancel := context.WithCancel(ctxx)
	defer cancel()

	tracer := otel.GetTracerProvider().Tracer("client")
	_, span := tracer.Start(ctx, "client_caller")
	defer span.End()

	time.Sleep(600 * time.Millisecond)
	// ================================================

	reply, err := c.SayHello(ctx, &kangaroo.Request{Name: "hello"})
	if err != nil {
		println(err.Error())
	}
	println(reply.Message)

	//app.Wait()
}

```
