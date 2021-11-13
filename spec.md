
# EDL design notes
## 0. Intro
```
a := "hey";
b := so Object; /* spawnobject experssion */
b.prop = b;
b@"prop" == b;
```

## 1. Type system
### 1.1 Primitive types
```
/* Values with no propreties */
null, undef;

/* bool, number, string, symbol */
Bool, Number, String, Symbol;

/* Primitive values */
true, false;
nan, inf, 3.14;
"ascii only"; 

/* Note: this is object, not bool */
b := so Bool();
/* and these are primitive values with immutable properties */
b = Bool(); b = true; 
```

### 1.2 Complex types, Functions
Prototype inheritance for Object and Function instances
```
                                       For user types
Instance    prototype property: .pro,  set by so operator
Constructor prototype property: .init, set by runtime
Constructor link      property: .ctor, set by runtime

Object.init, Function.init are mutable objects with immutable .pro

(so).pro----(Object.init).pro----(null)
             .ctor  |  |
		       |    |  |---------|
		       |    |            |
		       |  .init        .pro   
		      (Object).pro----(Function.init)
                               .ctor   |   |
                                 |     |   |
                                 |     |   |
                                 |   .init |
                                (Function).pro
Constructors:   Object, Function
Instance types: object, function
```
Property access operators
```
o := so Object;
o.x      = 1;  /* by name */
o@"x"   == 1;  /* by value */

o@22    = 66;  /* by index */
o@"22" == 66;  /* by value */

sym := Symbol();
o@sym;         /* by symbol */
```
## 2. Language syntax
```
stmt_list   : stmt | stmt_list stmt
block_stmt  : '{' stmt_list '}'
stmt        : expr_stmt | block_stmt | decl_stmt | control ';'
            | fn_flow expr_stmt
            | 'if' '(' expr ')' stmt
            | 'if' '(' expr ')' stmt 'else' stmt
            | 'for' '(' expr_stmt expr_stmt expr ')' stmt
            | 'for' '(' decl_stmt expr_stmt expr ')' stmt
            | 'for' '(' id 'of' expr ')' stmt
            | 'try' expr_stmt 'catch' '(' id ')' expr_stmt
            | 'import' '{' id_list '}' from str ';'
            | 'export' '{' id_list '}' ';'

control     : continue | break
fn_flow     : return | yield | throw 

decl_stmt   : decl_list ';'
decl_list   : decl | decl_list ',' decl
decl        : id ':=' expr

expr_stmt   : asgn_expr ';' | ';'

asgn_expr   : e_lor  | un_expr '='  asgn_expr
e_lor       : e_land | e_lor   '-'  e_land
e_land      : e_bor  | e_land  '||' e_bor
e_bor       : e_xor  | e_bor   '|'  e_xor
e_xor       : e_band | e_xor   '^'  e_band
e_band      : e_eq   | e_band  '&'  e_eq
e_eq        : e_rel  | e_eq    '==' e_rel | e_eq '===' e_rel
e_rel       : e_sh   | e_rel   '>'  e_sh  | e_rel 'instanceof'
e_sh        : e_add  | e_sh    '>>' e_add
e_add       : e_mul  | e_add   '+'  e_mul
e_mul       : e_un   | e_un    '*'  e_un

e_un        : e_post | '!' e_un | 'so' e_un | 'delete' e_un | 'typeof' e_un
e_post      : e_prim
            | e_post '.' e_prim
            | e_post '@' e_prim
            | so '.' id
            | e_post '++'
            | 'so' e_post '(' args ')'
            | 'fn' '(' args ')' block_stmt
            | 'co' '(' args ')' block_stmt

e_prim      : id | num | str
            | this | null | undef | true | false | nan | inf
```
## 3. Language features
### 3.1 Basic operations
```
sum := 0;
for (i := 0; i < 10; i++)
    sum = sum + i * i;
```
### 3.2 Lambdas, scopes, lexical environment
Scopes: _block\_stmt_, _fn/co-expr_, _if/for/try/catch-body_, _for-entry_
```
x := 1;
{ x := 2 }
create_adder := fn(a) {
    return fn(x) { return a + x };
};
create_adder(3)(5) == true;

linc  := fn(x) { return x + 1; };
lzero := fn(f) { return fn(x) { return x; }; };
lone  := fn(f) { return fn(x) { return fn(x); }; };

ladd := fn(l1, l2) {
	return fn(f) {
		return fn(x) { return l1(f)(l2(f)(x)); };
	};
};
lmul = fn(l1, l2) {
	return fn(f) {
		return fn(x) { return l1(l2(f))(x); };
	};
};
lsix := lmul(ladd(lone, lone), ladd(lone, ladd(lone, lone)));
lsix(inc)(0);
```
### 3.3 User types
```
Car := fn(c) { this.color = c; };
// so.tgt  ~ called with so or not
// so.args ~ access arguments as array

Car.init.model = "Honda"; // all cars are Hondas
Car.init.getColor = fn() { return this.color };

car := so Car("red");
car.pro === Car.init;
car.getColor; // "red"
car instanceof Car; // true

car.driver = "Vova";
delete car.driver; // :(
```
### 3.4 Coroutines, Iterators
```
it_res := fn(a, b) { res := so Object; res.val = a; res.end = b; return res };

// via closure
make_rangeit_fn := fn(n) {
    return fn() { n--; return it_res(n, n < 0); };
};
it := so Object;
it.go = make_rangeit_fn(3);
it.go(); it.go(); it.go(); it.go();

// via object: do it yourself

// via coroutines
make_rangeit := fn(n) {
    return co(x) {
        for (i = x; i < 10; ++i)
            yield it_res(i, false);
        yield it_res(-1, true);
    }; // reaching end is similar to "return undef;"
}

iterable_obj := so Object;
// Symbol.genit defined in any iterable object
iterabel_obj@(Symbol.genit) = fn() { return make_rangeit(5); };

for(v in iterbale_obj)
    // do smth
```
### 3.4 Exceptions
```
throws := fn() { throw "meow"; };

try {
    throws;
} catch (e) {
    // do something;
}
```
## 4. Standard library
### 4.0 Required fetures
```
Function.init.call
Function.init.apply
...
```
### 4.1 stdio
### 4.2 String
### 4.3 Array
### 4.4 Future
```
Future(fn);
Future.handle(ok_handle, fail_handle);
Future.wait();
```