## Golang迷宫算法输出


maze.in
```
8,8
0 1 0 1 0 0 0 0
0 0 0 0 0 1 1 0
1 1 1 1 0 1 0 0
0 0 0 1 1 0 0 1
0 1 0 0 0 0 1 0
0 0 1 1 1 0 0 0
1 0 0 0 1 1 1 1
0 0 1 0 0 0 0 0
```



```go
package algorithm

import (
	"fmt"
	"os"
)

type point struct {
	i,j int
}

// 返回二维切片 解析结构
func readMaze(filename string) [][] int {
	var row, col int
	file, e := os.Open(filename)
	if e != nil {
		panic(e)
	}
	fmt.Fscanf(file, "%d,%d", &row, &col)
	maze := make([][]int, row)
	for i := range maze {
		maze[i] = make([] int, col)
		for j := range maze[i] {
			fmt.Fscanf(file, "%d", &maze[i][j])
		}
	}
	return maze
}

// 不同方向上走一步
var dirs = [4] point {
	{-1, 0}, {0, -1}, {1, 0}, {0, 1},
}

// 相当于运算符重载 计算在走一步后的位置
func (p point) add (r point) point {
	return point{p.i + r.i, p.j + r.j}
}

// return int 下一步
// bool 表示下一步是不是有值
func (p point) at (grid [][] int) (int, bool)  {
	// 如果 值小于 0 就会越界不在maze中
	// 如果 值大于边界 grid 就是 往下 越界
	if p.i < 0 || p.i >= len(grid) {
		return 0, false
	}
	// 如果i行没有问题 那么查看是否j列是否越界
	if p.j < 0 || p.j >= len(grid[p.i]) {
		return 0, false
	}
	return grid[p.i][p.j], true
}

func walk(maze[][] int, start point, end point) [][]int {
	steps := make([][] int, len(maze))
	for i := range steps {
		steps[i] = make([] int, len(maze[i]))
	}

	// 前进步伐队列
	Q := []point{start}
	for len(Q) > 0 {
		cur := Q[0]
		Q = Q[1:]

		// 发现了终点
		if cur == end {
			break
		}

		for _, dir := range dirs {
			next := cur.add(dir)
			// maze at next is 0 迷宫的下一步是0
			// steps at next is 0 步伐的下一步是0 ???
			// next != start 下一步不等于开始
			val, ok := next.at(maze)
			// 如果有越界 或 撞到了边界内部的墙
			if !ok || val == 1{
				continue
			}

			// ??? steps 迷宫的copy对象?
			val, ok = next.at(steps)
			if !ok || val != 0 {
				continue
			}

			// 如果返回到了出口 那么 continue
			if next == start {
				continue
			}

			curSteps, _ := cur.at(steps)
			// 步伐填充入队列
			steps[next.i][next.j] = curSteps + 1
			// next点加入队列
			Q = append(Q, next)

		}
	}

	return steps
}

func RunMaze(fileName string) {
	maze := readMaze(fileName)
	steps := walk(maze, point{0, 0}, point{len(maze) - 1, len(maze[0]) - 1})
	for _, row := range steps {
		for _, val := range row {
			fmt.Printf("%3d", val)
		}
		fmt.Println()
	}
}
```