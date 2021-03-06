# [vicsy/dev](https://github.com/codr4life/vicsydev) | consing Forth
posted Feb 6th 2017, 05:00 pm

### preramble
In a previous [post](https://github.com/codr4life/vicsydev/blob/master/lispy_forth.md), I presented the humble beginnings of [Lifoo](https://github.com/codr4life/lifoo); a Lispy, Forth-like language implemented in Common Lisp. This post goes further into specific features and the reasoning behind them. I decided from the start that this was going to be a fresh take on Forth, in the spirit of Lisp; taking nothing for granted; and I ran into plenty of interesting choices as a result.

### repl
If you wan't to play along with the examples, a basic REPL may be started by cloning the [repository](https://github.com/codr4life/lifoo), followed by loading and evaluating ```(lifoo:lifoo-repl)```.

```
CL-USER> (lifoo:lifoo-repl)
Welcome to Lifoo,
press enter on empty line to evaluate,
exit ends session

Lifoo> 1 2 +

3

Lifoo> exit

NIL
CL-USER>
```

### reader
One of the goals set early on in the design process was to reuse the Lisp reader for reading Lifoo code. Looking back, sticking with this choice was fundamental to achieving a seamless integration since it acted as a natural obstacle to deviating from the Lisp way.

```
Lifoo> "1 2 +" read

(1 2 +)

Lifoo> "1 2 +" read eval

3

Lifoo> (1 2 +) write

"1 2 +"

Lifoo> (1 2 +) write read

(1 2 +)

Lifoo> (1 2 +) write read eval

3
```

### quoting
Lifoo treats all list literals as quoted. When evaluating a list literal, the parser treats items as code tokens. The price for convenience is not being able to evaluate items in list literals without mapping eval or building from scratch, but the approach fits like a glove with the simplicity of Forth and plays nice with Lisp. ```inline``` pre-compiles the preceding list down to a Lisp lambda.

```
Lifoo> (1 2 3)

(1 2 3)

Lifoo> (1 2 3) (2 *) map

(2 4 6)

Lifoo> (2 *) inline

#<FUNCTION {1008CD673B}>

Lifoo> (1 2 3) (2 *) inline map

(2 4 6)

Lifoo> ((1 2 +) (3 4 +) (5 6 +)) (eval) map

(3 7 11)

Lifoo> (1 2 +) inline
       (3 4 +) inline
       (5 6 +) inline
       stack reverse
       (eval) map

(3 7 11)
```

### symbols
Since Forth doesn't use a special call syntax; symbols in the token stream are interpreted as words, functions calls. Luckily, Common Lisp offers another kind of symbols in the form of keywords. In Lifoo; regular symbols are evaluated as words, while keywords are treated as symbols.

```
Lifoo> "lifoo" symbol

:LIFOO

Lifoo> :lifoo

:LIFOO
```

### comparisons
Lisp reserves common operators for use with numbers, ```+-*/<>``` and more; Lifoo follows this tradition but also provides generic compare operators that work for numbers as well as any other Lifoo values.

```
Lifoo> 1 2 + 3 =

T

Lifoo> "abc" "def" neq?

T

Lifoo> "def" "abc" lt?

T

Lifoo> '(1 2 3) '(1 2 3 4) cmp

1
```

### setf
The beauty of ```setf``` is that it untangles specifying a place from setting its value. If you still can't see it; imagine writing a generic function that can set indexes in arrays and replace tails of lists in any other language; then add fields in structs and keys in hash tables; ```setf``` allows you to pull tricks like that without missing a beat; and on top of that you can hook your own places into the protocol. Lifoo provides a comparable ```set``` word that sets values for any preceding stack cell that's hooked in.

```
Lifoo> #(1 2 3) 1 nth 4 set drop

#(1 4 3)

Lifoo> clear :bar var 42 set env

((:BAR . 42))


(define-lisp-word :set ()
  (let* ((val (lifoo-pop))
         (cell (lifoo-peek-cell))
         (set (lifoo-set cell)))
    (unless set
      (error "missing set: ~a" val))
    (funcall set val)))
```

### del
One thing Python got right (despite missing the ```setf``` train) was providing an extendable protocol for deletion. Separating concerns into pieces of generic functionality like this is what enables exponential power gains. Lifoo provides a ```del``` word that works like ```set``` but deletes places instead.

```
Lifoo> (1 2 3) 1 nth del drop

(1 3)

Lifoo> "abc" 1 nth del drop

"ac"


(define-lisp-word :del ()
  (let* ((cell (lifoo-peek-cell))
         (val (lifoo-val cell))
         (del (lifoo-del cell)))
    (unless del
      (error "missing del: ~a" val))
    (funcall del)
    (lifoo-push val)))
```

### definitions
I decided to deviate from the traditional Forth syntax for defining words, since neither the Lisp reader nor I approve of that level of cleverness. Lifoo provides a ```define``` word that defines preceding code and symbol as a word. ```define``` can be called anywhere; overwrites any previous bindings, and makes the new definition available for immediate use.

```
Lifoo> (drop drop 42) :+ define
       1 2 +

42

Lifoo> :+ source

(DROP DROP 42)
```

### packages
Package systems always seem to get in the way sooner or later, every language comes with it's own set of arbitrary limitations. Lifoo provides an extensible, tag-based init protocol. Words may belong to several different packages, and importing any of them imports the word. All tags in an init block must be matched for words to be imported. Besides support for ```init```; packages are second class, and the only way of defining one is from Lisp. the good news is that Lisp is right around the corner as long as you remembered to load the ```meta``` package; and the REPL loads all packages by default.

```
Lifoo> (define-lifoo-init (:foo :bar)
         (define-word :baz () 39 +))
       lisp eval
       (:foo :bar) init
       3 baz

42


(define-macro-word :lisp (in out)
  (cons (cons in
            `(lifoo-push (lambda ()
                           (lifoo-optimize)
                           ,(first (first out)))))
      (rest out))))
      
      
(defmacro define-lifoo-init (tags &body body)
  "Defines init for TAGS around BODY"
  `(setf (gethash ',tags *lifoo-init*)
         (lambda (exec)
           (with-lifoo (:exec exec)
             ,@body))))
```

### structs
A programming language doesn't get far without the ability to define new types from within the language. Lifoo provides a simple but effective interface to defstruct. Outside of Lifoo the struct is anonymous to not clash with existing Lisp definitions. Words are automatically generated for ```make-foo```, ```foo-p``` and fields with setters when the ```struct``` word is evaluated.

```
Lifoo> ((bar -1) baz) :foo struct
       nil make-foo foo?

T

Lifoo> (:bar 42) make-foo
       foo-bar

42

Lifoo> (:bar 42) make-foo
       foo-bar 43 set
       foo-bar

43


(define-lisp-word :struct ()
  (let ((fields (lifoo-pop))
        (name (lifoo-pop)))
    (define-lifoo-struct name fields)))


(defmacro define-lifoo-struct (name fields)
  "Defines struct NAME with FIELDS"
  `(progn
     (let ((lisp-name (gensym))
           (fs ,fields))
       (eval `(defstruct (,lisp-name)
                ,@fs))
       (define-lifoo-struct-fn
           (keyword! 'make- ,name) (symbol! 'make- lisp-name)
         (lifoo-pop))
       (define-lifoo-struct-fn
           (keyword! ,name '?) (symbol! lisp-name '-p)
         (list (lifoo-peek)))
       (dolist (f fs)
         (let ((fn (if (consp f) (first f) f)))
           (define-lifoo-struct-fn
               (keyword! ,name '- fn) (symbol! lisp-name '- fn)
             (list (lifoo-peek)) :set? t))))))

(defmacro define-lifoo-struct-fn (lifoo lisp args &key set?)
  "Defines word LIFOO that calls LISP with ARGS"
  (with-symbols (_fn _sfn)
    `(let ((,_fn (symbol-function ,lisp))
           (,_sfn (and ,set? (fdefinition (list 'setf ,lisp)))))
       
       (define-lisp-word ,lifoo ()
         (lifoo-push
          (apply ,_fn ,args)
          :set (when ,set?
                 (lambda (val)
                   (lifoo-pop)
                   (funcall ,_sfn val (lifoo-peek)))))))))
```

### macros
I went back and Forth a couple of times on the issue of macros; once token streams come on silver plates, it's really hard to resist the temptation of going all in. What I ended up with is essentially Lisp macros with a touch of Forth. Like Lisp, Lifoo macros operate on streams of tokens. But since Forth is post-fix; macros deal with previously parsed, rather than wrapped, tokens. Lifoo provides macro words that are called to translate the token stream when code is parsed. A token stream consists of pairs of tokens and generated code, and the result of a macro call replaces the token stream from that point on. 

```
Lifoo> :+ macro?

NIL

Lifoo> (+ 1 2) compile

(PROGN (LIFOO-CALL '+) (LIFOO-PUSH 1) (LIFOO-PUSH 2))

Lifoo> :inline macro?

T

(+ 1 2) inline compile

(PROGN
  (FUNCALL #<FUNCTION {100426B4EB}>))

(define-macro-word :assert (in out)
  (cons (cons in
            `(let ((ok? (progn
                          ,@(lifoo-compile (first (first out)))
                          (lifoo-pop))))
               (unless ok?
                 (lifoo-error "assert failed: ~a"
                              ',(first (first out))))))
        (rest out)))


(defmacro define-macro-word (id (in out &key exec)
                             &body body)
  "Defines new macro word NAME in EXEC from Lisp forms in BODY"
  `(lifoo-define ,id (make-lifoo-word
                      :id ,id
                      :macro? t
                      :fn (lambda (,in ,out)
                            ,@body))
                 :exec (or ,exec *lifoo*)))
```

### always, throw & catch
One of the features that was waiting for macros to arrive was throwing and catching. ```always``` is implemented as a macro that wraps the entire token stream in ```unwind-protect```, catch pulls the same trick with ```handler-case```, and ```throw``` signals a condition. If a thrown value isn't caught, an error is reported.

```
Lifoo> :frisbee throw
       "skipped" print ln
       (:always) always
       (drop) catch

:ALWAYS

Lifoo> :up throw
       "skipped" print ln
       (:caught cons) catch

(:CAUGHT . :UP)


(define-lisp-word :throw ()
  (lifoo-throw (lifoo-pop)))

(define-macro-word :always (in out)
  (list
    (cons in `(unwind-protect
                (progn
                  ,@(reverse (mapcar #'rest (rest out))))
             (lifoo-eval ',(first (first out)))))))

(define-macro-word :catch (in out)
  (list
    (cons in `(handler-case
               (progn
                 ,@(reverse (mapcar #'rest (rest out))))
             (lifoo-throw (c)
               (lifoo-push (value c))
               (lifoo-eval ',(first (first out)))))))))
```

### multi-threading
All Lifoo code runs in a ```lifoo-exec``` object, the result of accessing a ```lifoo-exec``` from multiple threads at the same time is undefined. Spawning new threads clones the current exec and [channels](http://vicsydev.blogspot.de/2017/01/channels-in-common-lisp.html) are used for communication.

```
Lifoo> 1 chan 42 send recv

42

Lifoo> 0 chan 
       (1 2 + send :done) 1 spawn swap 
       recv swap drop swap 
       wait cons

(:DONE . 3)


(define-lisp-word :spawn ()
  (let* ((num-args (lifoo-pop))
       (expr (lifoo-pop))
       (exec (make-lifoo
              :stack (clone
                      (if num-args
                          (subseq (stack *lifoo*) 0 num-args)
                          (stack *lifoo*)))
              :words (clone (words *lifoo*))))
       (thread (make-thread (lambda ()
                              (lifoo-eval expr :exec exec)))))
    (lifoo-push thread)))
```

### performance
The only thing I can say for sure so far is that it's slower than Lisp, yet fast enough for my needs. And it should be; since most code is pre-compiled all the way down to Lisp lambdas. The reason structs are slower is that defining a struct in Lisp with accessors is a complex operation. Evaluating ```(cl4l-test:run-suite '(:lifoo) :reps 3)``` after loading will give you an idea of the speed on your setup by running all tests 3 x 30 times.

```
(cl4l-test:run-suite '(:lifoo) :reps 3)
(lifoo abc)                   0.072
(lifoo array)                 0.032
(lifoo compare)               0.012
(lifoo env)                   0.012
(lifoo error)                 0.036
(lifoo flow)                  0.272
(lifoo io)                      0.0
(lifoo list)                  0.044
(lifoo log)                     0.0
(lifoo meta)                   0.08
(lifoo stack)                 0.008
(lifoo string)                0.024
(lifoo struct)                1.072
(lifoo thread)                0.084
(lifoo word)                  0.048
TOTAL                         1.796
NIL
```

There is much left to be said, but this post needs to end somewhere. You may find more in the same spirit [here](http://vicsydev.blogspot.de/) and [here](https://github.com/codr4life/vicsydev), and a full implementation of this idea and more [here](https://github.com/codr4life).

peace, out
