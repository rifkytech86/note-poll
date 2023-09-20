# Pooling

- Request & Response 

- Short pooling 
  set client send INTERVAL, like ajax call interval

- long polling 
  set backend/server hold waiting once finish time waiting send response

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
