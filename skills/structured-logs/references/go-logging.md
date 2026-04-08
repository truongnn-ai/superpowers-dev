# Go Logging Setup (zerolog + chi/stdlib)

## Complete file listing

### go.mod additions
```
require github.com/rs/zerolog v1.33.0
require github.com/google/uuid v1.6.0
```

### pkg/logging/logger.go
```go
package logging

import (
	"context"
	"os"
	"github.com/rs/zerolog"
)

type traceKey struct{}

var Logger zerolog.Logger

func Init() {
	level, _ := zerolog.ParseLevel(os.Getenv("LOG_LEVEL"))
	if level == zerolog.NoLevel {
		level = zerolog.InfoLevel
	}

	Logger = zerolog.New(os.Stdout).
		Level(level).
		With().
		Timestamp().
		Str("service", os.Getenv("SERVICE_NAME")).
		Logger()
}

func WithTraceID(ctx context.Context, traceID string) context.Context {
	return context.WithValue(ctx, traceKey{}, traceID)
}

func TraceID(ctx context.Context) string {
	if tid, ok := ctx.Value(traceKey{}).(string); ok {
		return tid
	}
	return "no-trace"
}

func Ctx(ctx context.Context) zerolog.Logger {
	return Logger.With().Str("trace_id", TraceID(ctx)).Logger()
}
```

### pkg/middleware/tracing.go
```go
package middleware

import (
	"net/http"
	"time"
	"github.com/google/uuid"
	"yourapp/pkg/logging"
)

func Tracing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		traceID := r.Header.Get("X-Trace-ID")
		if traceID == "" {
			traceID = uuid.NewString()
		}

		ctx := logging.WithTraceID(r.Context(), traceID)
		log := logging.Ctx(ctx)

		start := time.Now()
		log.Info().
			Str("method", r.Method).
			Str("path", r.URL.Path).
			Msg("request_started")

		wrapped := &responseWriter{ResponseWriter: w, status: 200}
		w.Header().Set("X-Trace-ID", traceID)

		next.ServeHTTP(wrapped, r.WithContext(ctx))

		log.Info().
			Str("method", r.Method).
			Str("path", r.URL.Path).
			Int("status", wrapped.status).
			Int64("duration_ms", time.Since(start).Milliseconds()).
			Msg("request_completed")
	})
}

type responseWriter struct {
	http.ResponseWriter
	status int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.status = code
	rw.ResponseWriter.WriteHeader(code)
}
```

### pkg/httpclient/client.go (outgoing calls with trace propagation)
```go
package httpclient

import (
	"context"
	"fmt"
	"net/http"
	"time"
	"yourapp/pkg/logging"
)

var client = &http.Client{Timeout: 10 * time.Second}

func Do(ctx context.Context, method, url string) (*http.Response, error) {
	log := logging.Ctx(ctx)
	traceID := logging.TraceID(ctx)

	req, err := http.NewRequestWithContext(ctx, method, url, nil)
	if err != nil {
		return nil, err
	}
	req.Header.Set("X-Trace-ID", traceID)

	log.Info().Str("target", url).Str("method", method).Msg("outgoing_call_started")
	start := time.Now()

	resp, err := client.Do(req)
	duration := time.Since(start).Milliseconds()

	if err != nil {
		if ctx.Err() != nil {
			log.Error().
				Str("error_category", "timeout").
				Str("error_detail", fmt.Sprintf("%s %s timed out after %dms", method, url, duration)).
				Msg("outgoing_call_timeout")
		} else {
			log.Error().
				Str("error_category", "connection").
				Str("error_detail", fmt.Sprintf("Failed to connect to %s: %s", url, err.Error()[:200])).
				Msg("outgoing_call_connection_failed")
		}
		return nil, err
	}

	log.Info().
		Str("target", url).
		Int("status", resp.StatusCode).
		Int64("duration_ms", duration).
		Msg("outgoing_call_completed")

	return resp, nil
}
```

### main.go (wiring)
```go
package main

import (
	"net/http"
	"os"
	"yourapp/pkg/logging"
	"yourapp/pkg/middleware"
)

func main() {
	logging.Init()

	mux := http.NewServeMux()
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.Write([]byte(`{"status":"ok"}`))
	})

	handler := middleware.Tracing(mux)
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	logging.Logger.Info().
		Str("port", port).
		Str("environment", os.Getenv("ENVIRONMENT")).
		Msg("service_started")

	http.ListenAndServe(":"+port, handler)
}
```
