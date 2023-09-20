# Pooling

- Request & Response 

- Short pooling 
  set client send INTERVAL, like ajax call interval

- long polling 
  set backend/server hold waiting once finish time waiting send response

  example: 
  - once message empty, server will send once message not empty
```
var (
	message     string
	subscribers = make(map[chan string]struct{})
	mu          sync.Mutex
)

func main() {
	http.HandleFunc("/getmessage", getMessageHandler)
	http.HandleFunc("/updatemessage", updateMessageHandler)

	go func() {
		for {
			//time.Sleep(5 * time.Second)
			mu.Lock()
			if message != "" {
				for ch := range subscribers {
					ch <- message
				}
				message = ""
			}
			mu.Unlock()
		}
	}()

	http.ListenAndServe(":8050", nil)
}

func getMessageHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/plain")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")

	ch := make(chan string)
	mu.Lock()
	subscribers[ch] = struct{}{}
	mu.Unlock()

	select {
	case msg := <-ch:
		fmt.Fprint(w, msg)
		mu.Lock()
		delete(subscribers, ch)
		close(ch)
		mu.Unlock()
	case <-time.After(30 * time.Second): // Set a timeout of 30 seconds
		mu.Lock()
		delete(subscribers, ch)
		close(ch)
		mu.Unlock()
		fmt.Fprint(w, "No new messages")
	}
}

func updateMessageHandler(w http.ResponseWriter, r *http.Request) {
	newMessage := r.URL.Query().Get("message")
	if newMessage != "" {
		mu.Lock()
		message = newMessage
		mu.Unlock()
	}
}

```

- server send event 
  client send request to backend/server, backend will send until when queu success 

```
func main() {

	r := gin.Default()

	r.GET("/events", func(c *gin.Context) {
		c.Header("Content-Type", "text/event-stream")
		c.Header("Cache-Control", "no-cache")
		c.Header("Connection", "keep-alive")

		messageChan := make(chan string)
		defer close(messageChan)

		go func() {
			for i := 0; i < 5; i++ {
				messageChan <- fmt.Sprintf("data: Message %d\n\n", i)
				time.Sleep(2 * time.Second)
			}
		}()

		for {
			select {
			case message := <-messageChan:
				c.Writer.WriteString(message)
				c.Writer.Flush()
			case <-c.Request.Context().Done():
				return
			}
		}
	})

	r.Run(":8050")
}
```
