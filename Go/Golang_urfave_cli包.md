## Golang Command 参数

常用NewApp command 参数
```
github.com/urfave/cli/v2 v2.2.0
```

```go
func CommandTest()  {
	app := cli.NewApp()
	app.Name = "TestTodo"
	app.Usage = "fight the loneliness!"
	app.Action = func(c *cli.Context) error {
		fmt.Println("Hello Friend")
		return nil
	}
	app.Run(os.Args)
}
```

