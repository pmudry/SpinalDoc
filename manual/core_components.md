# `spinal.core` components
The core components of the language are described in this document. It is part of the general [Spinal user guide](userGuide.md).

##Clock Domain
In Spinal, clock and reset signals can be combined to create a clock domain. Clock domain could be applied to some area of the design, then synchronous elements instantiated into this area will then use this clock domain implicitly.
It's permitted to have inner clock domain area.

ClockDomain(clock : Bool[,reset : Bool[,enable : Bool]]])

```scala
val coreClock = Bool
val coreReset = Bool
val coreClockDomain = ClockDomain(coreClock,coreReset)
...
val coreArea = new ClockingArea(coreClockDomain){
  val coreClockedRegister = Reg(UInt(4 bit))
}
```
###Clock configuration
Additionally, following elements of each clock domain are configurable via a ClockDomainConfig class :

| Property | Possibilities |
| ------- | ---- |
| clockEdge | RISING, FALLING |
| ResetKind | ASYNC, SYNC |
| resetActiveHigh | true, false |
| clockEnableActiveHigh| true, false |

By default, a ClockDomain is applied to the whole design. The configuration of this one is :
- clock : rising edge
- reset: asynchronous, active high
- no enable signal

###Cross Clock Domain
Spinal check that there is no unwanted/unspecified cross clock domain read. If you want to read a signals that is emited by another ClockDomain area, you should add the `crossClockDomain` tag to the destination signal.

```scala
val asynchronousSignal = UInt(8 bit)
...
val buffer0 = Reg(UInt(8 bit)).addTag(crossClockDomain)
val buffer1 = Reg(UInt(8 bit))
buffer0 := asynchronousSignal
buffer1 := buffer0   //Second register stage to be metastability safe

//Or in less lines :
val buffer0 = RegNext(asynchronousSignal).addTag(crossClockDomain)
val buffer1 = RegNext(buffer0) 
```	

## Assignements
There is multiple assignment operator :

| Symbole| Description |
| ------- | ---- |
| := | Standard assignment, equivalent to '<=' in VHDL/Verilog <br> last assignment win, value updated at next delta cycle  |
| /= | Equivalent to := in VHDL and = in Verilog <br> value updated instantly |
| <> |Automatic connection between 2 signals. Direction is inferred by using signal direction (in/out) <br> Similar behavioural than :=  |

```scala
//Because of hardware concurrency is always read with the value '1' by b and c
val a,b,c = UInt(4 bit)
a := 0
b := a
a := 1  //a := 1 win
c := a  

var x = UInt(4 bit)
val y,z = UInt(4 bit)
x := 0
y := x      //y read x with the value 0
x \= x + 1
z := x      //z read x with the value 1
```
Spinal check that bitcount of left and right assignment side match. There is multiple ways to adapt bitcount of BitVector (Bits, UInt, SInt) :

| Resizing ways | Description|
| ------- | ---- |
| x := y.resized | Assign x wit a resized copy of y, resize value is automatically inferred to match x  |
| x := y.resize(newWidth) | Assign x with a resized copy of y, size is manually calculated |

There are 2 cases where spinal automaticly resize things :
| Assignement | Problem | Spinal action|
| ------- | ---- |
| myUIntOf_8bit := U(3) | U(3) create an UInt of 2 bits, which don't match with left side  | Because  U(3) is a "weak" bit inferred signal, Spinal resize it automatically |
| myUIntOf_8bit := U(2 -> False default -> true) | The right part infer a 3 bit UInt, which doesn't match with the left part | Spinal reapply the default value to bit that are missing |

## Conditional assignment
As VHDL and Verilog, wire and register can be conditionally assigned by using when and switch syntaxes
```scala
when(cond1){
  //execute when      cond1 is true
}.elsewhen(cond2){
  //execute when (not cond1) and cond2
}.otherwise{
  //execute when (not cond1) and (not cond2)
}

switch(x){
  is(value1){
    //execute when x === value1
  }
  is(value2){
    //execute when x === value2
  }
  default{
    //execute if none of precedent condition meet
  }
}
```

## Component/Hierarchy
Like in VHDL and Verilog, you can define components that could be used to build a design hierarchy.  But unlike them, you don't need to bind them at instantiation.

```scala
class AdderCell extends Component {
  //Declaring all in/out in an io Bundle is probably a good practice
  val io = new Bundle { 
    val a, b, cin = in Bool
    val sum, cout = out Bool
  }
  //Do some logic
  io.sum := io.a ^ io.b ^ io.cin
  io.cout := (io.a & io.b) | (io.a & io.cin) | (io.b & io.cin)
}

class Adder(width: Int) extends Component {
  ...
  //Create 2 AdderCell
  val cell0 = new AdderCell
  val cell1 = new AdderCell
  cell1.io.cin := cell0.io.cout //Connect carrys
  ...
  val cellArray = Array.fill(width)(new AdderCell) 
  ...
}
```

Syntax to define in/out is the following :

| Syntax | Description| Return
| ------- | ---- | --- |
| in/out(x : Data) | Set x an input/output | x |
| in/out Bool | Create an input/output Bool | Bool |
| in/out Bits/UInt/SInt[(x bit)]| Create an input/output of the corresponding type | T|

There is some rules about component interconnection :
- Components can only read outputs/inputs signals of children components
- Components can read outputs/inputs ports values
- If for some reason, you need to read a signals from far away in the hierarchy (debug, temporal patch) you can do it by using the value returned by some.where.else.theSignal.pull().

##Area
Sometime, creating a component to define some logic is overkill and to much verbose. For this kind of cases you can use Area :

```scala
class UartCtrl extends Component {
  ...
  val timer = new Area {
    val counter = Reg(UInt(8 bit))
    val tick = counter === 0
    counter := counter - 1
    when(tick) {
      counter := 100
    }
  }
  val tickCounter = new Area {
    val value = Reg(UInt(3 bit))
    val reset = False
	when(timer.tick) {          // Refer to the tick from timer area
      value := value + 1
    }
    when(reset) {
      value := 0
    }
  }
  val stateMachine = new Area {
    ...
  }
}
```
##Function
The ways how you can use scala function to generate hardware are radically different than VHDL/Verilog for some reason :
- You can instanciate register, combinatorial and component inside them.
- You don't have to play with process/@alwas that limit the scope of assignement of signals
- Everything work by reference, which allow many manipulation.



## Compile

```scala
// spinal.core contain all basics (Bool, UInt, Bundle, Reg, Component, ..)
import spinal.core._

//A simple component definition
class MyTopLevel extends Component {
  //Define some input/output. Bundle like a VHDL record or a verilog struct.
  val io = new Bundle {
    val a = in Bool
    val b = in Bool
    val c = out Bool
  }

  //Define some asynchronous logic
  io.c := io.a & io.b
}

//This is the main of the project. It create a instance of MyTopLevel and
//call the SpinalHDL library to flush it into a VHDL file.
object MyMain {
  def main(args: Array[String]) {
    SpinalVhdl(new MyTopLevel)
  }
}
```

##Memory

| Syntax | Description|
| ------- | ---- |
| Mem(type : Data,size : Int) |  Create a RAM |
| Mem(type : Data,initialContent : Array[Data]) |  Create a ROM    |

| Syntax | Description| Return |
| ------- | ---- | --- |
| mem(x) |  Asynchronous read | T |
| mem(x) := y |  Synchronous write | |
| mem.readSync(address,enable) | Synchronous read | T|
 
##Instanciate VHDL and Verilog IP
 In some cases, it could be usefull to instanciate a VHDL or a Verilog component into a Spinal design. To do that, you need to define BlackBox which is like a Component, but its internal implementation should be provided by a separate VHDL/Verilog file to the simulator/synthesis tool.
 
```scala
class Ram_1w_1r(_wordWidth: Int, _wordCount: Int) extends BlackBox {
  val generic = new Generic {
    val wordCount = _wordCount
    val wordWidth = _wordWidth
  }

  val io = new Bundle {
    val clk = in Bool

    val wr = new Bundle {
      val en = in Bool
      val addr = in UInt (log2Up(_wordCount) bit)
      val data = in Bits (_wordWidth bit)
    }
    val rd = new Bundle {
      val en = in Bool
      val addr = in UInt (log2Up(_wordCount) bit)
      val data = out Bits (_wordWidth bit)
    }
  }

  mapClockDomain(clock=io.clk)
}
```
##Utils
The Spinal core contain some utils :

| Syntax | Description| Return |
| ------- | ---- | --- |
| log2Up(x : BigInt) | Return the number of bit needed to represent x states | Int |
| isPow2(x : BigInt) | Return true if x is a power of two | Boolean|

Much more tool and utils are present in spinal.lib

##Some example

```scala
class Counter(width : Int) extend Component{
  val io = new Bundle{
    val clear = in Bool
    val value = out UInt(width bit)
  }
  val register = Reg(UInt(width bit)) init(0)
  register := register + 1
  when(io.clear){
    register := 0
  }
  io.value := register
}
```

```scala
class CarryAdder(size : Int) extends Component{
  val io = new Bundle{
    val a = in UInt(size bit)
    val b = in UInt(size bit)
    val result = out UInt(size bit)      //result = a + b
  }

  var c = False                   //Carry, like a VHDL variable
  for (i <- 0 until size) {
    //Create some intermediate value in the loop scope.
    val a = io.a(i)  
    val b = io.b(i)  

    //The carry adder's asynchronous logic
    io.result(i) := a ^ b ^ c
    c = (a & b) | (a & c) | (b & c);    //variable assignment
  }
}


object CarryAdderProject {
  def main(args: Array[String]) {
    SpinalVhdl(new CarryAdder(4))
  }
}
```
#spinal.lib


##Stream interface
The Stream interface is a simple handshake protocol to carry payload. They could be used for example to push and pop elements into a FIFO, send requests to a UART controller, etc.

| Signal | Driver| Description | Don't care when
| ------- | ---- | --- |  --- |
| valid | Master | When high => payload present on the interface  | |
| payload| Master | Content of the transaction | valid is low |
| ready| Slave | When low => transaction are not consumed by the slave | valid is low |

| Syntax | Description| Return | Latency |
| ------- | ---- | --- |  --- |
| Stream(type : Data) | Create a Stream of a given type | Stream[T] | |
| master/slave Stream(type : Data) | Create a Stream of a given type <br> Initialized with corresponding in/out setup | Stream[T] |
| x.queue(size:Int) | Return a Stream connected to x through a FIFO | Stream[T] | 2 |
| x.m2sPipe() | Return a Stream drived by x <br>through a register stage that cut valid/data paths | Stream[T] |  1 |
| x.s2mPipe() | Return a Stream drived by x <br> ready paths is cut by a register stage | Stream[T] |  0 |
| x << y <br> y >> x | Connect y to x | | 0 |
| x <-< y <br> y >-> x | Connect y to x through a m2sPipe  |   | 1 |
| x <&#47;< y <br> y >&#47;> x | Connect y to x through a s2mPipe|   | 0 |
| x <-/< y <br> y >&#47;-> x | Connect y to x through s2mPipe().m2sPipe() <br> => no combinatorial path between x and y |  | 1 |
| x.haltWhen(cond : Bool) | Return a Stream connected to x <br> Halted when cond is true | Stream[T] | 0 |
| x.throwWhen(cond : Bool) | Return a Stream connected to x <br> When cond is true, transaction are dropped | Stream[T] | 0 |

Examples :
```scala
class StreamFifo[T <: Data](dataType: T, depth: Int) extends Component {
  val io = new Bundle {
    val push = slave Stream (dataType)
    val pop = master Stream (dataType)
  }
  ...
}

class StreamArbiter[T <: Data](dataType: T,portCount: Int) extends Component {
  val io = new Bundle {
    val inputs = Vec(slave Stream (dataType),portCount)
    val output = master Stream (dataType)
  }
  ...
}
```

The following code will create this logic :
<img src="https://cdn.rawgit.com/SpinalHDL/SpinalDoc/master/asset/picture/stream_throw_m2spipe.svg"   align="middle" width="300">

```scala
case class RGB(channelWidth : Int) extends Bundle{
  val red   = UInt(channelWidth bit)
  val green = UInt(channelWidth bit)
  val blue  = UInt(channelWidth bit)

  def isBlack : Bool = red === 0 && green === 0 && blue === 0
}

val source = Stream(RGB(8))
val sink   = Stream(RGB(8))
sink <-< source.throwWhen(source.payload.isBlack)
```


##Flow interface