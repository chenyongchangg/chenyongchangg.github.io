```Go
package main

import (
	"fmt"
	"sync"
)

type CountWithLock struct {
	Value int
	Lock  *sync.Mutex
}

func main() {
	counter := CountWithLock{
		Value: 100,
		Lock:  &sync.Mutex{},
	}
	waitg := sync.WaitGroup{}
	waitg.Add(100)
	msgChan := make(chan int)
	go func() {
		waitg.Wait()
		close(msgChan)
	}()
	for i := 0; i < 4; i++ {
		go func() {
			for {
				counter.Lock.Lock()
				if counter.Value > 0 {
					counter.Value -= 1
					msgChan <- 1
				}
				counter.Lock.Unlock()
			}

		}()
	}

	for v := range msgChan {
		fmt.Print(v)
		waitg.Done()
	}
}

```