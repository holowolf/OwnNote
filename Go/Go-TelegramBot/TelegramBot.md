## TelegramBot

code
```
package telegram_bot_api

import (
	telegramBotApi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
	"github.com/hpcloud/tail"
	"log"
)

const ChatId = "-1001441017850"

var Message = make(chan string, 0)

func ReadFileToMessage()  {
	go func() {
		telegramBotMessage()
	}()

	file, err := tail.TailFile("./test", tail.Config{Follow: true})
	if err != nil {
		panic(err)
	}
	for line := range file.Lines {
		Message <- line.Text
	}
}

func telegramBotMessage() {
	for {
		message := <- Message
		bot, err := telegramBotApi.NewBotAPI("1325662411:AAH7KqmXsh8BH8yW574xfUXLlmo-gwG8sAE")
		if err != nil {
			log.Panic(err)
		}
		bot.Debug = true

		channel := telegramBotApi.NewMessageToChannel(ChatId, message)
		if _, err = bot.Send(channel); err != nil {
			log.Printf("err = %v", err)
		}
	}
}
```

go mod
```
module github.com/yeenughu/study_all

go 1.14

require (
	github.com/go-telegram-bot-api/telegram-bot-api/v5 v5.0.0-rc1
	// github.com/afex/hystrix-go v0.0.0-20180502004556-fa1af6a1f4f5
	// github.com/coreos/etcd v3.3.9+incompatible
	// github.com/go-telegram-bot-api/telegram-bot-api/v5 v5.0.0-rc1
	// github.com/gogo/protobuf v1.3.1 // indirect
	// github.com/golang/protobuf v1.4.3 // indirect
	github.com/hpcloud/tail v1.0.0
	gopkg.in/fsnotify.v1 v1.4.7 // indirect
	gopkg.in/tomb.v1 v1.0.0-20141024135613-dd632973f1e7 // indirect
)

```