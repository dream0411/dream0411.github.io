---
layout: default
title: Stop sending to chan in multiple senders and reader in Golang
---

{{ page.title }}
===

I post a topic in [golang-nuts](https://groups.google.com/forum/#!topic/golang-nuts/4sfNgjgOBMY) about how to stop multiple concurrent sending to one chan and read all element in this chan with multiple readers. After that I found a solution.

```
package main

import (
	"cloud-base/util"
	"fmt"
	"log"
	"sync"
	"sync/atomic"
	"time"
)

var (
	ErrClosed = fmt.Errorf("worker closed")
	OkCount   int64
	FailCount int64

	SendOnClosedCount int64

	kReaderAndSenderCount = 50
)

type someTask struct {
}

func (s someTask) Delete() {
}

type Worker struct {
	quitCh      chan struct{}
	taskCh      chan someTask
	taskNew     int64
	taskDone    int64
	wg          sync.WaitGroup
	senderCount sync.WaitGroup
}

func NewWorker() *Worker {
	w := &Worker{
		quitCh:      make(chan struct{}),
		taskCh:      make(chan someTask, 128),
		wg:          sync.WaitGroup{},
		senderCount: sync.WaitGroup{},
	}
	return w
}

func (w *Worker) run(waitAllSender bool) {
	for {
		select {
		case <-w.quitCh:
			if waitAllSender {
				w.senderCount.Wait()
			}
			for empty := false; !empty; {
				select {
				case t := <-w.taskCh:
					atomic.AddInt64(&w.taskDone, 1)
					t.Delete()
				default:
					empty = true
					break
				}
			}
			return

		case t := <-w.taskCh:
			//if !ok {
			//	break
			//}
			atomic.AddInt64(&w.taskDone, 1)
			t.Delete()
		}
	}
}

func (w *Worker) Send(t someTask) error {
	//defer func() {
	//	if e := recover(); e != nil {
	//		atomic.AddInt64(&SendOnClosedCount, 1)
	//	}
	//}()
	var err error
	w.senderCount.Add(1)
	select {
	case <-w.quitCh:
		err = ErrClosed

	case w.taskCh <- t:
		atomic.AddInt64(&w.taskNew, 1)
	}
	w.senderCount.Done()
	return err
}

func (w *Worker) Stop() {
	close(w.quitCh)
	//close(w.taskCh)
}

func test() {
	w := NewWorker()

	// many producers
	for i := 0; i < kReaderAndSenderCount; i++ {
		w.wg.Add(1)
		go func() {
			defer w.wg.Done()
			for {
				if err := w.Send(someTask{}); err != nil {
					break
				}
			}
		}()
	}

	// many consumers
	for i := 0; i < kReaderAndSenderCount; i++ {
		w.wg.Add(1)
		go func(waitAllSender bool) {
			defer w.wg.Done()
			w.run(waitAllSender)
		}(i == 0)
	}

	// stopped at some time
	go func() {
		time.Sleep(3 * time.Second)
		w.Stop()
	}()

	w.wg.Wait()

	taskNewCount := atomic.LoadInt64(&w.taskNew)
	taskDoneCount := atomic.LoadInt64(&w.taskDone)
	if taskNewCount != taskDoneCount {
		log.Printf("Ending, tasks left count: %d, chan count: %d\n", taskNewCount-taskDoneCount, len(w.taskCh))
		atomic.AddInt64(&FailCount, 1)
	} else {
		atomic.AddInt64(&OkCount, 1)
	}
}

func main() {
	util.BeginProfCPU()
	util.BeginProfMem()
	begin := time.Now()
	wg := sync.WaitGroup{}
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			test()
		}()
	}
	wg.Wait()
	end := time.Now()
	err := util.EndProfMem()
	if err != nil {
		log.Printf("mem prof error: %v", err)
	}
	err = util.EndProfCPU()
	if err != nil {
		log.Printf("cpu prof error: %v", err)
	}
	log.Printf("time: %v, %d ok, %d failed, %d panic on send", end.Sub(begin), atomic.LoadInt64(&OkCount), atomic.LoadInt64(&FailCount), atomic.LoadInt64(&SendOnClosedCount))
}
```

I don't want to use "defer recover()" to recover from panic of "sending to closed chan" because it may recover other panic in real world code. So I don't close the taskCh chan.

Main implemention is here:

1. Add quitCh in Worker, when Stop() was called, quitCh would be closed, and this signal should be seen by all readers and followed senders;

2. Add a sync.WaitGroup member senderCount to record current senders, every call to Send will call senderCount.Add(1) before trying select quitCh and sending to chan, and senderCount.Done() after select block;

3. Add senderCount.Wait() in one special reader goroutine; when this reader was signaled by quitCh, it will Wait until all senders were done. After this Wait, no more element would be sent into taskCh because followed calls of Send would select <-quitch. Then this special reader will process all left elements in taskCh chan.
