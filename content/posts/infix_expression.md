+++
title = "Infix Expression"
date = "2023-03-04T14:07:18+08:00"
tags = ["algorithm"]
keywords = ["two stack", "infix expression"]
description = "Notes of the Two Stack Algorithm for evaluating infix expression"
showFullContent = false
readingTime = false
hideComments = false
+++

# Infix Expression

Infix expression are the arithmatical expressions we learned how to evaluate back in primary school, for example,
```
1 + 2 * (3 + 4)
```
is a typical infix expression.

Besides infix expressions, there are other forms of arithmatical expressions, for example, Poland expressions. Instead of putting the operator between operands, in Poland expressions, operator is placed before two operands.

Following is a Poland expression, in LISP style.
```
(+ (* 2 (+ 3 4)) 1)
```

Also, there is reverse Poland expression, which place the operator behind two operands
```
((((3 4 +) 2 *) 1 +)
```

Reverse Poland expression is actually the most suitable form for computers, as it resembles how today's stack-based machine instructions work:
```
mov $3 %rax    ; store 3 in register rax
mov $4 %rbx    ; store 4 in rbx
add %rax %rbx  ; add value in rbx to rax
mov $2 %rbx
mul %rax %rbx
mov $1 %rbx
add %rax %rbx
```

It would be wonderful if everyone use reverse Poland expressions, but this is not how the world works. In real world, computers serve people. But who serve the computers? Of course we programmers.
```
        serve             serve
People <------ Computers <------ Programmer
```


## Two Stack Algorithm

A way to evaluate infix expressions is to use two seperate stacks, one for operands and one for operators.

We traverse through the expression, and whenever we encounter a operand (a float, integer, whatever), we push it into the operand stack $S_v$. Likewise, whenever we encounter a operator (add, substract, multiply, divide, power, whatever), we need to decided whether to push it into the operator stack $S_o$, or to **reduce** $S_o$ by popping some operators and apply them to operands in $S_v$.

The tricky part is making the decision based on occurrance of parentheses, combination direction and operator priority. To handle these annoying parentheses, we introduce two set of priorities: in-stack priority $P_1(x)$ and out-stack priority $P_2(x)$, where $x$ is the underlying operator.

$P_1(x)$ is defined as (left to right, low to high)
```
(, + -, * /, ^
```
and $P_2(x)$ is defined as
```
+ -, * /, ^, (
```
The only difference is the priority of left parentheses `(`. It has high out-stack priority so that it will always be pushed into stack. On the other hand, it has low in-stack priority so that successing operators can be pushed into stack. Note that the right parentheses `)` is never pushed into stack.

We annotate the encountered operator $x$, and the last operator in the stack $x_0$, the last two operands in stack $a$, $b$.

- If $P_2(x) > P_1(x_0)$, push $x$ and do nothing. This is because there may be a higher-priority operator remaining in the expression stream, which need to be applied first, so current one has to wait.
- If $P_2(x) = P_1(x_0)$, reduce operator stack $S_o$ once.
- If $P_2(x) < P_1(x_0)$, reduce operator stack $S_o$ until a `(` is the top item in stack. This is because according to the rule of pushing operators, the priorities of operators from the top `(` to `x_0` strictly ascend, so we can safely combine them from right to left (from stack top to bottom).

## Converting to Poland and reverse Poland expressions

Using Two Stack Algorithm, we can also convert infix expressions to other two forms. We simply need to redefine what "operators" and "operands" are.

More specifically, operands are now strings, and operators format operand strings in a certain way. For example, for Poland expression, the logic can be expressed with

```python
def add(a: PolandExpr, b: PolandExpr) -> PolandExpr:
    return f"(+ {a} {b})"
```

And similiarly, for reverse Poland expression,

```python
def mul(a: RevPolandExpr, b: RevPolandExpr) -> RevPolandExpr:
    return f"({a} {b} *)"
```

## Rust Implementation

Following is a well-documented, robust and unit-tested Rust implementation of the algorithm. It can handle several cases of bad expressions.


{{< code language="rust" title="lib.rs" id="1" expand="Show" collapse="Hide" >}}

    mod expr {
        /// Wraps a Poland expression
        #[derive(Debug, PartialEq)]
        pub struct PolandExpr(pub String);

        impl From<String> for PolandExpr {
            fn from(value: String) -> Self {
                Self(value)
            }
        }

        impl From<f64> for PolandExpr {
            fn from(value: f64) -> Self {
                Self(value.to_string())
            }
        }

        impl std::fmt::Display for PolandExpr {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                write!(f, "{}", self.0)
            }
        }

        /// Wraps a reverse Poland expression
        #[derive(Debug, PartialEq)]
        pub struct RevPolandExpr(pub String);

        impl<'a> From<String> for RevPolandExpr {
            fn from(value: String) -> Self {
                Self(value)
            }
        }

        impl From<f64> for RevPolandExpr {
            fn from(value: f64) -> Self {
                Self(value.to_string())
            }
        }

        impl std::fmt::Display for RevPolandExpr {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                write!(f, "{}", self.0)
            }
        }
    }

    mod marker {
        use super::expr::{PolandExpr, RevPolandExpr};
        use super::op::Op;

        /// Marks a type that is the output of the calculator
        pub trait ResultT: std::fmt::Debug + std::fmt::Display + From<f64> {
            fn applied(self, op: &Op<Self>, rhs: Self) -> Self;
        }

        /// Output [`f64`]: the value of the expression
        impl ResultT for f64 {
            fn applied(self, op: &Op<Self>, rhs: Self) -> Self {
                use Op::*;
                match op {
                    Add => self + rhs,
                    Sub => self - rhs,
                    Mul => self * rhs,
                    Div => self / rhs,
                    Pow => self.powf(rhs),
                    _ => panic!("Can not apply a parentheses"),
                }
            }
        }

        /// Output [`PolandExpr`]: convert the expression to Poland expression
        impl ResultT for PolandExpr {
            /// Apply an [`Op`] to `a` and `b`, returning (`op` `a` b`)
            fn applied(self, op: &Op<Self>, rhs: Self) -> Self {
                PolandExpr(format!("({} {} {})", op.to_char(), self.0, rhs.0))
            }
        }

        /// Output [`RevPolandExpr`]: convert the expression to reverse Poland
        /// expression
        impl ResultT for RevPolandExpr {
            /// Apply an [`Op`] to `a` and `b`, returning (`a` `b` `op`)
            fn applied(self, op: &Op<Self>, rhs: Self) -> Self {
                RevPolandExpr(format!("({} {} {})", self.0, rhs.0, op.to_char()))
            }
        }
    }

    mod op {
        use super::marker::ResultT;

        pub trait OpApply<T> {
            fn apply(&self, a: T, b: T) -> T;
        }

        /// The operations supported by this calculator
        #[derive(Clone, Copy, PartialEq, Eq, Debug)]
        pub enum Op<T: ResultT> {
            /// Addition
            Add,
            /// Subtraction
            Sub,
            /// Multiply
            Mul,
            /// Division
            Div,
            /// Power
            Pow,
            /// Left parentheses
            LP,
            /// Right parentheses
            RP,
            _Phantom(T),
        }

        impl<T> Op<T>
        where
            T: ResultT,
        {
            /// The priority of each operator, if it is in the stack.
            /// Priority determines whether a reduction will be triggered,
            /// or whether a operator will be pushed
            pub fn in_stack_priority(&self) -> i32 {
                use Op::*;
                match self {
                    Add => 1,
                    Sub => 1,
                    Mul => 2,
                    Div => 2,
                    Pow => 3,
                    LP => 0,
                    RP => -1,
                    _ => panic!("_Phantom should never be instantiated!"),
                }
            }

            /// The priority of each operator, if it is outside the stack
            pub fn out_stack_priority(&self) -> i32 {
                use Op::*;
                match self {
                    Add => 1,
                    Sub => 1,
                    Mul => 2,
                    Div => 2,
                    Pow => 3,
                    LP => 99,
                    RP => -1,
                    _ => panic!("_Phantom should never be instantiated!"),
                }
            }

            /// Parse a [`Op`] from a [`char`]. Returns [`None`] on failure.
            pub fn parse(s: char) -> Option<Self> {
                use Op::*;
                match s {
                    '+' => Some(Add),
                    '-' => Some(Sub),
                    '*' => Some(Mul),
                    '/' => Some(Div),
                    '^' => Some(Pow),
                    '(' => Some(LP),
                    ')' => Some(RP),
                    _ => None,
                }
            }

            pub fn to_char(&self) -> char {
                use Op::*;
                match self {
                    Add => '+',
                    Sub => '-',
                    Mul => '*',
                    Div => '/',
                    Pow => '^',
                    LP => '(',
                    RP => ')',
                    _ => panic!("_Phantom should never be instantiated!"),
                }
            }

            pub fn is_right_parentheses(&self) -> bool {
                if let Op::RP = self {
                    true
                } else {
                    false
                }
            }

            pub fn is_left_parentheses(&self) -> bool {
                if let Op::LP = self {
                    true
                } else {
                    false
                }
            }
        }

        impl<T> OpApply<T> for Op<T>
        where
            T: ResultT,
        {
            fn apply(&self, a: T, b: T) -> T {
                a.applied(self, b)
            }
        }
    }

    mod error {
        /// Indicates an error happened in evaluation
        #[derive(Debug)]
        pub enum Error {
            /// A string slice can not be interpreted into an operand (float) nor an operator
            UnknownSymbol(String),
            /// Too many right parentheses are in the expression
            TooManyRP,
            /// Not enough operands are in the expression
            InsufficientOperands,
            /// Too many left parentheses are in the expression
            TooManyLP,
        }

        impl std::fmt::Display for Error {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                match self {
                    Error::UnknownSymbol(sym) => write!(f, "Unknown Symbol {}", sym),
                    Error::TooManyRP => write!(f, "Too many right parentheses"),
                    Error::InsufficientOperands => write!(f, "Insufficient operands"),
                    Error::TooManyLP => write!(f, "Too many left parentheses"),
                }
            }
        }
    }

    mod token {
        use std::marker::PhantomData;

        use super::error::Error;
        use super::marker::ResultT;
        use super::op::Op;

        /// Either an operator or a value
        #[derive(Debug)]
        pub enum Token<T>
        where
            T: ResultT,
        {
            Op(Op<T>),
            Val(f64),
        }

        /// An iterator over string slice, yielding [`Token`]
        pub struct Tokens<'a, T> {
            s: &'a str,
            buf: String,
            _phantom: PhantomData<T>,
        }

        impl<'a, T> Tokens<'a, T>
        where
            T: ResultT,
        {
            pub fn new(s: &'a str) -> Self {
                Self {
                    s,
                    buf: String::new(),
                    _phantom: PhantomData,
                }
            }

            /// Try to parse whatever inside `self.buf` as a [`f64`], and return it
            /// wrapped inside a [`Token`]. A [`Error::UnknownSymbol`] is thrown if
            /// it cannot be parsed.
            fn dump_buf(&mut self) -> Result<Token<T>, Error> {
                return self
                    .buf
                    .parse::<f64>()
                    .map_err(|_| {
                        let sym = std::mem::take(&mut self.buf);
                        Error::UnknownSymbol(sym)
                    })
                    .map(|x| {
                        self.buf.clear();
                        Token::Val(x)
                    });
            }
        }

        impl<'a, T> Iterator for Tokens<'a, T>
        where
            T: ResultT,
        {
            type Item = Result<Token<T>, Error>;
            fn next(&mut self) -> Option<Self::Item> {
                let mut chars = self.s.chars();

                while let Some(next_char) = chars.next() {
                    if next_char == ' ' {
                        self.s = &self.s[next_char.len_utf8()..];
                        continue;
                    }

                    match Op::parse(next_char) {
                        None => {
                            self.buf.push(next_char);
                            self.s = &self.s[next_char.len_utf8()..]
                        }

                        Some(op) => {
                            if self.buf.len() == 0 {
                                // No non-operator symbol encountered
                                self.s = &self.s[next_char.len_utf8()..];
                                return Some(Ok(Token::Op(op)));
                            } else {
                                // Some non-operator symbols encountered, try parse them into value
                                return Some(self.dump_buf());
                            }
                        }
                    }
                }

                if self.buf.len() == 0 {
                    None
                } else {
                    Some(self.dump_buf())
                }
            }
        }
    }

    pub use error::Error;
    pub use expr::{PolandExpr, RevPolandExpr};
    pub use marker::ResultT;
    pub use op::Op;
    use op::OpApply;
    pub use token::{Token, Tokens};

    /// The calculator
    /// The behavior of the calculator depends on `T`.
    /// - If `T` is [`f64`], the calculator evaluates the value of the expression
    /// - If `T` is [`PolandExpr`] or [`RevPolandExpr`], the calculator converts
    ///   the expression to Poland and reverse Poland expression respectively.
    /// Spaces in the expression are ignored.
    ///
    /// ```
    /// use calculator::*;
    /// let mut calc = Calculator::<f64>::new();
    /// let out = calc.eval("1 + 2").unwrap();
    /// assert!((out - 3.0).abs() < 1e-6);
    /// ```
    /// ```
    /// use calculator::*;
    /// let mut calc = Calculator::<PolandExpr>::new();
    /// let out = calc.eval("1+2").unwrap();
    /// assert_eq!(out.0, "(+ 1 2)");
    /// ```
    pub struct Calculator<T>
    where
        T: ResultT,
    {
        op_stack: Vec<Op<T>>,
        val_stack: Vec<T>,
    }

    impl<T> Calculator<T>
    where
        T: ResultT,
    {
        /// Create a new calculator
        pub fn new() -> Self {
            Self {
                op_stack: vec![Op::LP],
                val_stack: Vec::new(),
            }
        }

        /// Reset the internal state. Used for recovering from error.
        /// Using this instead of creating a new instance reduces memory allocation.
        pub fn reset(&mut self) {
            self.op_stack.clear();
            self.val_stack.clear();
            self.op_stack.push(Op::LP);
        }

        /// Evaluate an expression
        pub fn eval(&mut self, expr: &str) -> Result<T, Error> {
            let mut token_stream = Tokens::new(expr);

            while let Some(token) = token_stream.next() {
                let token = token?;

                match token {
                    Token::Val(val) => self.val_stack.push(val.into()),
                    Token::Op(op) => self.process_op(op)?,
                };
            }

            // pretending that there is a trailing RP, to match the LP put initially in op_stack
            self.reduce_op_stack()?;

            // As operands are delimitated using operators, there can't be too many operands
            // (But there can be less, which throws InsufficientOperands)
            if self.val_stack.len() != 1 {
                panic!(
                    "val_stack should have only one item, but now val_stack={:?}",
                    self.val_stack
                );
            }

            if self.op_stack.len() != 0 {
                return Err(Error::TooManyLP);
            }

            // put back the initial LP for next evaluation
            self.op_stack.push(Op::LP);

            Ok(self.val_stack.pop().unwrap())
        }

        /// Pop an operator from op_stack and apply it to last-two operands in
        /// val_stack, pushing the result back into the val_stack.
        fn reduce_op_stack_once(&mut self) -> Result<Option<()>, Error> {
            let top_op = self
                .op_stack
                .pop()
                .map_or(Err(Error::TooManyRP), |x| Ok(x))?;

            if top_op.is_left_parentheses() {
                return Ok(None);
            }

            let operand_r = self
                .val_stack
                .pop()
                .map_or(Err(Error::InsufficientOperands), |x| Ok(x))?;
            let operand_l = self
                .val_stack
                .pop()
                .map_or(Err(Error::InsufficientOperands), |x| Ok(x))?;

            self.val_stack.push(top_op.apply(operand_l, operand_r));

            Ok(Some(()))
        }

        /// Repeatedly pop operator from op_stack and apply it. Stop when a LP
        /// is encountered. The LP is consumed.
        fn reduce_op_stack(&mut self) -> Result<(), Error> {
            loop {
                if self.reduce_op_stack_once()?.is_none() {
                    break;
                }
            }

            Ok(())
        }

        fn process_op(&mut self, op: Op<T>) -> Result<(), Error> {
            // op_stack should at least have a LP; If no operators are left in the
            // stack but operators are still feeding in, it must be that there are
            // too many RPs
            let top_op = self
                .op_stack
                .last()
                .map_or(Err(Error::TooManyRP), |x| Ok(x))?;

            if op.is_right_parentheses() {
                self.reduce_op_stack()?;
            } else if op.out_stack_priority() > top_op.in_stack_priority() {
                self.op_stack.push(op);
            } else if op.out_stack_priority() < top_op.in_stack_priority() {
                self.reduce_op_stack()?;
                // the LP is consumed by reduce_op_stack. It must be added back as
                // the corresponding RP isn' reached yet.
                self.op_stack.push(Op::LP);
                self.op_stack.push(op);
            } else {
                // in-stack priority equals out-stack
                self.reduce_op_stack_once()?;
                self.op_stack.push(op);
            }

            Ok(())
        }
    }

    #[cfg(test)]
    mod test {
        use super::*;

        #[test]
        fn test_token() {
            let s = "1+(3*4/5)^6";
            let tokens: Vec<_> = Tokens::new(s).collect();
            let tokens: Vec<Token<f64>> = tokens.into_iter().map(|x| x.unwrap()).collect();

            use Op::*;
            use Token::Op as TOp;
            use Token::Val;
            let should_be = vec![
                Val(1.0),
                TOp(Add),
                TOp(LP),
                Val(3.0),
                TOp(Mul),
                Val(4.0),
                TOp(Div),
                Val(5.0),
                TOp(RP),
                TOp(Pow),
                Val(6.0),
            ];

            println!("{:?}", tokens);
            assert_eq!(tokens.len(), should_be.len());
            for (tok, ans) in tokens.iter().zip(should_be.iter()) {
                match (tok, ans) {
                    (TOp(a), TOp(b)) => assert_eq!(a, b),
                    (Val(a), Val(b)) => assert!((a - b).abs() < 1e-6),
                    _ => panic!("Not equal. tokens={:?}", tokens),
                }
            }
        }

        #[test]
        #[should_panic]
        fn test_token_err() {
            let s = "ab";
            let tokens: Vec<_> = Tokens::new(s).collect();
            let _: Vec<Token<f64>> = tokens.into_iter().map(|x| x.unwrap()).collect();
        }

        fn assert_ans(val: f64, ans: f64) {
            if (val - ans).abs() > 1e-6 {
                panic!("output={}, but ans={}", val, ans);
            }
        }

        fn cf() -> Calculator<f64> {
            Calculator::new()
        }
        fn cp() -> Calculator<PolandExpr> {
            Calculator::new()
        }

        #[test]
        fn test_eval() {
            assert_ans(cf().eval("1+2").unwrap(), 3.0);
            assert_ans(cf().eval("1*2+3/1-2^2").unwrap(), 1.0);
            assert_ans(cf().eval("2*2*(4-3)+8").unwrap(), 12.0);
            assert_ans(cf().eval("3+2*(4-3*2/2+4)*(1+2)").unwrap(), 33.0);
            assert_ans(cf().eval("2*((1+2)/3+2*(4-3))-2^(3-2)").unwrap(), 4.0);
            assert_ans(cf().eval("8-1-2").unwrap(), 5.0);
            assert_ans(cf().eval("12/2/2").unwrap(), 3.0);
            assert_ans(cf().eval("1+2*(1*5-2-2*1)+5").unwrap(), 8.0);
        }

        #[test]
        fn test_err() {
            let out = cf().eval("1+");
            if let Err(Error::InsufficientOperands) = out {
            } else {
                panic!("out should be insufficient operands error")
            };

            let out = cf().eval("1+2)");
            if let Err(Error::TooManyRP) = out {
            } else {
                panic!("out should be too many RP error")
            };
        }

        #[test]
        fn test_to_poland() {
            assert_eq!(cp().eval("1+2").unwrap(), "(+ 1 2)".to_string().into());
            assert_eq!(
                cp().eval("1*2+3/1-2^2").unwrap(),
                "(- (+ (* 1 2) (/ 3 1)) (^ 2 2))".to_string().into()
            );
            assert_eq!(
                cp().eval("2*2*(4-3)+8").unwrap(),
                "(+ (* (* 2 2) (- 4 3)) 8)".to_string().into()
            );
        }
    }

{{< /code >}}

With a simple commandline interface:

{{< code language="rust" title="main.rs" id="2" expand="Show" collapse="Hide" >}}

    use indoc::indoc;
    use pico_args::Arguments;
    use std::ffi::OsString;
    use std::io::{stdin, stdout, Write};

    use calculator::Calculator;
    use calculator::PolandExpr;
    use calculator::ResultT;
    use calculator::RevPolandExpr;

    /// REPL mode
    fn repl<T: ResultT>(mut calc: Calculator<T>) {
        let mut buf = String::new();

        loop {
            print!("> ");
            stdout().flush().unwrap();
            stdin().read_line(&mut buf).unwrap();

            match calc.eval(&buf.trim()) {
                Err(e) => {
                    println!("! {}", e);
                    calc.reset();
                }
                Ok(val) => println!("= {}", val),
            }

            buf.clear();
        }
    }

    /// Evaluate and Print
    fn ep<T: ResultT>(mut calc: Calculator<T>, exprs: Vec<&str>) {
        let mut exit_flag = 0;

        for expr in exprs {
            match calc.eval(expr) {
                Err(e) => {
                    println!("! {}", e);
                    calc.reset();
                    exit_flag += 1;
                }
                Ok(val) => println!("{}", val),
            }
        }

        std::process::exit(exit_flag);
    }

    fn finish<T: ResultT>(calc: Calculator<T>, left_args: Vec<OsString>) {
        if left_args.len() == 0 {
            repl(calc);
        } else {
            ep(
                calc,
                left_args.iter().map(|x| x.to_str().unwrap()).collect(),
            );
        }
    }

    static HELP: &str = indoc! {"
    Usage: %prog [-h] [-p|-r] [exprs..]

    Options
        -h       : Show this message
        -p       : Calculate Poland expression
        -r       : Calculate reverse Poland expression
        Otherwise: Evaluate the expression

    Arguments
        A list of expressions. If expressions are provided, they will be evaluated
        in succession, and the result will be printed line-by-line. If one expression
        is bad, the corresponding line will start with '!', followed with a error
        message.

        If no expression is provided, REPL mode will be activated.

    Exit Codes
        0     : All good
        Other : If not in REPL mode, indicates the number of bad expressions
    "};

    fn main() {
        let mut args = Arguments::from_env();

        if args.contains("-h") {
            println!("{HELP}");
            std::process::exit(0);
        } else if args.contains("-p") {
            let calc = Calculator::<PolandExpr>::new();
            finish(calc, args.finish());
        } else if args.contains("-r") {
            let calc = Calculator::<RevPolandExpr>::new();
            finish(calc, args.finish());
        } else {
            let calc = Calculator::<f64>::new();
            finish(calc, args.finish());
        }
    }

{{< /code >}}

The interface depends on `indoc = "2.0.0"` and `pico-args = "0.5.0"`.


