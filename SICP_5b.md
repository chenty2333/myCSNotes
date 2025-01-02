# 关于SICP 5b 计算对象中，数字电路dsl(domain-sepcific language) lisp程序

这节课的难度还是比较大的，课堂上professor的语速和节奏很快。

把整个程序拆分成很多块，单独每一小块是比较容易理解的。但是课堂上的快节奏，让我很难在短时间里像拼拼图一样把整个程序的逻辑组合在一起。能大致解程序的过程，但是不懂为什么这样设计message-passing等模式。



让我们先来看看这个程序吧（以下代码使用racket-fmt格式化）：

```lisp
; a domain-specific language of digital circuits

#lang racket

;;;; PART 1: Core Infrastructure ;;;;

;; The agenda is our simulation timeline - it keeps track of what should happen when
(define the-agenda (make-parameter null))  ; Starts empty
(define (make-agenda) null)                ; Creates a new empty agenda
(define current-time 0)                    ; Keeps track of simulation time

;; Helper function to add actions to our timeline
(define (after-delay delay action)
  (add-to-agenda! (+ delay current-time) action (the-agenda)))

;; Add an action to the agenda at the specified time
(define (add-to-agenda! time action agenda)
  (the-agenda (cons (cons time action) agenda)))

;; Execute all pending actions in our timeline
(define (propagate)
  (if (null? (the-agenda))
      'done
      (let ([first-item (car (the-agenda))])
        (set! current-time (car first-item))
        (the-agenda (cdr (the-agenda)))
        ((cdr first-item))
        (propagate))))

;;;; PART 2: Wire Implementation ;;;;

;; Call each procedure in a list of procedures
(define (call-each procedures)
  (if (null? procedures)
      'done
      (begin
        ((car procedures))
        (call-each (cdr procedures)))))

; This is what a wire looks like under the hood
(define (make-wire)
  ; Each wire keeps track of two important things:
  (let ([signal-value 0] ; The current signal (0 or 1)
        [action-procedures '()]) ; List of procedures to notify

    ; This is called when we want to change the wire's value
    (define (set-my-signal! new-value)
      ; Only do something if the value actually changes
      (if (not (= signal-value new-value))
          (begin
            ; Update the stored value
            (set! signal-value new-value)
            ; Notify all connected components
            (call-each action-procedures))
          'done))

    ; This is how components "connect" to the wire
    (define (accept-action-procedure! proc)
      ; Add the new procedure to our notification list
      (set! action-procedures (cons proc action-procedures))
      ; Run it once immediately to initialize
      (proc))

    ; This is our message handler
    (define (dispatch m)
      (cond
        [(eq? m 'get-signal) signal-value]
        [(eq? m 'set-signal!) set-my-signal!]
        [(eq? m 'add-action!) accept-action-procedure!]
        [else (error "Unknown operation -- WIRE" m)]))
    dispatch))

;;;; PART 3: Interface Procedures ;;;;

;; Get the current value of a wire
(define (get-signal wire)
  (wire 'get-signal))

;; Set the value of a wire
(define (set-signal! wire new-value)
  ((wire 'set-signal!) new-value))

;; Add an action procedure to a wire
(define (add-action! wire action-procedure)
  ((wire 'add-action!) action-procedure))

;;;; PART 4: Basic Gates ;;;;

;; Define gate delays (you can adjust these)
(define inverter-delay 2)
(define and-gate-delay 3)
(define or-gate-delay 5)

;; Logical operations
(define (logical-not s)
  (if (= s 0) 1 0))

(define (logical-and s1 s2)
  (if (and (= s1 1) (= s2 1)) 1 0))

(define (logical-or s1 s2)
  (if (or (= s1 1) (= s2 1)) 1 0))

;; NOT gate (inverter)
(define (inverter input output)
  (define (invert-input)
    (let ([new-value (logical-not (get-signal input))])
      (after-delay inverter-delay (lambda () (set-signal! output new-value)))))
  (add-action! input invert-input)
  'ok)

;; AND gate
; First, let's look at how an AND gate connects to wires
(define (and-gate a1 a2 output)
  ; This procedure computes what the AND gate should do
  (define (and-action-procedure)
    ; Get current values from input wires
    (let ([new-value (logical-and (get-signal a1) (get-signal a2))])
      ; Schedule the output change after a delay
      (after-delay and-gate-delay (lambda () (set-signal! output new-value)))))

  ; Here's the key part: we register our procedure with both inputs
  (add-action! a1 and-action-procedure)
  (add-action! a2 and-action-procedure)
  'ok)

;; OR gate
(define (or-gate a1 a2 output)
  (define (or-action-procedure)
    (let ([new-value (logical-or (get-signal a1) (get-signal a2))])
      (after-delay or-gate-delay (lambda () (set-signal! output new-value)))))
  (add-action! a1 or-action-procedure)
  (add-action! a2 or-action-procedure)
  'ok)

;;;; PART 5: Debug Helper ;;;;

;; Monitor a wire by printing its changes
(define (probe name wire)
  (add-action! wire (lambda () (printf "~a ~a New-value = ~a~n" name current-time (get-signal wire))))
  'ok)

;;;; PART 6: Example Usage ;;;;

;; Initialize the agenda
(the-agenda (make-agenda))

;; Create some wires
(define input-1 (make-wire))
(define input-2 (make-wire))
(define output (make-wire))

;; Add a probe to watch the output
(probe 'output output)

;; Create an AND gate
(and-gate input-1 input-2 output)

;; Test the circuit
(set-signal! input-1 1)
(propagate)
(set-signal! input-2 1)
(propagate)
```

这里将程序分为多个部分，让我们逐个来看



### agenda是什么？

首先是基础设施部分：

```lisp
;; The agenda is our simulation timeline - it keeps track of what should happen when
(define the-agenda (make-parameter null))  ; Starts empty
(define (make-agenda) null)                ; Creates a new empty agenda
(define current-time 0)                    ; Keeps track of simulation time

;; Helper function to add actions to our timeline
(define (after-delay delay action)
  (add-to-agenda! (+ delay current-time) action (the-agenda)))

;; Add an action to the agenda at the specified time
(define (add-to-agenda! time action agenda)
  (the-agenda (cons (cons time action) agenda)))

;; Execute all pending actions in our timeline
(define (propagate)
  (if (null? (the-agenda))
      'done
      (let ([first-item (car (the-agenda))])
        (set! current-time (car first-item))
        (the-agenda (cdr (the-agenda)))
        ((cdr first-item))
        (propagate))))
```

关于代码部分不过多解释，来谈谈我们为什么需要一个`agenda`。

最最最开始`agenda`是如何被构建的？通过`make-agenda`。

我们用`make-agenda`来存储一个全局的时间表。之后所有待执行的操作，比如`set-signal!`后输入信号的变化。

但为什么`set-signal!`会引起`agenda`的变化呢？我们定位到`set-signal!`这个`procedure`的位置：

```lisp
;; Set the value of a wire
(define (set-signal! wire new-value)
  ((wire 'set-signal!) new-value))
```

诶，`set-signal!`传递了一个信号`'set-signal!`给我们的`wire`。

再来看看`wire`里面的`dispatch`：

```lisp
    ; This is our message handler
    (define (dispatch m)
      (cond
        [(eq? m 'get-signal) signal-value]
        [(eq? m 'set-signal!) set-my-signal!]
        [(eq? m 'add-action!) accept-action-procedure!]
        [else (error "Unknown operation -- WIRE" m)]))
    dispatch))
```

！！！麻烦，怎么这么多层调用(´。＿。｀)

`dispatch`再接收到`'set-signal!`信号之后，又调用了`set-my-signal!`这个`procedure`。

下面是关键点：

那`set-my-sigenl!`又又又做了什么？

```lisp
    ; This is called when we want to change the wire's value
    (define (set-my-signal! new-value)
      ; Only do something if the value actually changes
      (if (not (= signal-value new-value))
          (begin
            ; Update the stored value
            (set! signal-value new-value)
            ; Notify all connected components
            (call-each action-procedures))
          'done))
```

当`value`变化的时候，`signal-value`被赋值为`new-value`。

并且执行了一个`call-each`的过程！我们越来越接近真相了，我们定位到一切对`agenda`的修改都是`call-each`完成的，再来看看`call-each`做了什么？

```lisp
;; Call each procedure in a list of procedures
(define (call-each procedures)
  (if (null? procedures)
      'done
      (begin
        ((car procedures))
        (call-each (cdr procedures)))))
```

`call-each`只做了一件事，那就是遍历执行`procedures`里面注册的所有过程。

问题又来了，`procedures`里面的过程是谁注册的？

是`add-action!`，看代码！：

```lisp
;; Add an action procedure to a wire
(define (add-action! wire action-procedure)
  ((wire 'add-action!) action-procedure))
```

在`and-gate`里面的`add-action!`把`and-action-procedure`添加进了`wire`里的`procedures`。

（这里还有一个小点`new-value`的值是通过`logical-and`获得的）

```lisp
  (define (and-action-procedure)
    ; Get current values from input wires
    (let ([new-value (logical-and (get-signal a1) (get-signal a2))])
      ; Schedule the output change after a delay
      (after-delay and-gate-delay (lambda () (set-signal! output new-value)))))

  ; Here's the key part: we register our procedure with both inputs
  (add-action! a1 and-action-procedure)
  (add-action! a2 and-action-procedure)
  'ok)
```

当`call-each`遍历执行`procedures`的时候，在`and-gate`这个例子里面，其实最后执行到的就是这句话：

```lisp
(after-delay and-gate-delay (lambda () (set-signal! output new-value)))
```

aha! `after-delay`听起来就和`agenda`关系密切！

对于`agenda`的修改是通过`and-gate`的逻辑门中的`after-delay`完成的：`after-delay`调用了`add-to-agenda!`

我们再来看看`add-to-agenda!`：

```lisp
;; Add an action to the agenda at the specified time
(define (add-to-agenda! time action agenda)
  (the-agenda (cons (cons time action) agenda)))
```

aha! 就像剥洋葱一样，我们发现了`agenda`被修改的机制。

为什么`make-agenda`是第一个关键点呢？通过`agenda`的机制，我们其实已经了解了整个程序的大半。与、或、非门的具体实现逻辑是最容易理解的。

其实学到这里，我还是有点点小疑惑的，为什么？为什么要用这么复杂的一个机制，这种机制是如何被想出来的？

通过之前的学习，SICP体现了一种“建模”的思想，如果现实太难理解，那我我们就建立一个简单的模型吧！

“模型”本身的机制足够简单，我们很容易理解“模型”，但是如何将“模型”和“现实”连接在一起，理解其中机制是相对困难的。

老师上课举了一个例子“人有许多属性，比如体温、血压。如果课堂上有人尖叫，我的血压就会升高”，在真实世界中，这种信息的传播是直接的。我们能够看到或者听到或者仅仅只是感受到，“现实”传递给我们的信息。

但是在计算机不是物理世界，我们需要有一种建模方式把类似于“尖叫使血压升高”这样的“模式”映射到逻辑世界中。

前辈们构建了这样的模型，这种模型不是一种简单的二`if-else`判断，而是一个完善的，能涵盖许多情况的，类似物理世界的“法则”。或者说中文语境中的“道”。

“道生一，一生二，二生三，三生万物”，在此模型下，“物”与“物”可以传递信息，这就是“事件驱动的时间模型”。

还有种类似的模型`Happens-Before`，Leslie Lamport在他1978年的论文*Time, Clocks, and the Ordering of Events in a Distributed System*中说到：

> “We now introduce clocks into the system. We begin with an abstract point of view in which a clock is just a way of assigning a number to an event, where the number is thought of as the time at which the event occurred.”



好了，回到`agenda`，我们还有一个点没说，那就是`propagate`：

```lisp
;; Execute all pending actions in our timeline
(define (propagate)
  (if (null? (the-agenda))
      'done
      (let ([first-item (car (the-agenda))])
        (set! current-time (car first-item))
        (the-agenda (cdr (the-agenda)))
        ((cdr first-item))
        (propagate))))
```

`propagate`和`call-each`很像，不过`call-each`遍历的是`procedures`，而`propagate`遍历执行的是`the-agenda`。

当有`wire`的值发生变化时，修改`output`的过程被添加到了`the-agenda`中，直到调用`propagate`时再被执行。



以上，5b前半节课的笔记。