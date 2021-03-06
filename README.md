# nanoTimer
# Current Version - 0.3.0

![](https://api.travis-ci.org/Krb686/nanoTimer.png)

A much higher accuracy timer object that makes use of the node.js [hrtime](http://nodejs.org/api/process.html#process_process_hrtime) function call.

The nanotimer recreates the internal javascript timing functions with higher resolution.

##Note

- 1) With the normal timing functions, instead of dealing with the obscurities of multiple setTimeout and 
setInterval calls, now there is a concrete timer object, each of which can handle exactly 1 timeOut and 
setInterval task. This also means a reference is not needed to clear an interval since each timer object is
unique.

- 2) Timer objects use the non-blocking feature **setImmediate** for time counting and synchronization. This requires node v0.10.13 or greater

- 3) Errors in timing are also non-cumulative.  For example, when using the setInterval command, the timer counts
and compares the time difference since starting against the interval length specified and if it has run past the interval, 
it resets.  If the code had an error of 1 millisecond delay, the timer would actually count to 1001 milliseconds before resetting, and that
1 millisecond error would propagate through each cycle and add up very quickly!  To solve that problem, rather than resetting
the interval variable each cycle, it is instead incremented with each cycle count.  So on the 2nd cycle, it compares to 2000 milliseconds, 
and it may run to 2001.  Then 3000 milliseconds, running to 3001, and so on.  This is only limited by the comparison variable potentially overflowing, so I 
somewhat arbitrarily chose a value of 8 quadrillion (max is roughly 9 quadrillion in javascript) before it resets.  Even using nanosecond resolution however, 
the comparison variable would reach 8 quadrillion every 8 million seconds, or every 93.6ish days.

##Usage

```js

var NanoTimer = require('nanotimer');

var timerA = new NanoTimer();


```

Each NanoTimer object can run other functions that are already defined.  This can be done in 2 ways, either with a literal function object, or with a function declaration.

### By function object
```js
var NanoTimer = require('nanotimer');
var timerObject = new NanoTimer();


var countToOneBillion = function () {
    var i = 0;
    while(i < 1000000000){
        i++;
    }
};

var microsecs = timerObject.time(countToOneBillion, 'u');
console.log(microsecs);
```

or something like this:

### by function declaration

```js
var NanoTimer = require('nanotimer');

function main(){
    var timerObject = new NanoTimer();
    
    var microsecs = timerObject.time(countToOneBillion, 'u');
    console.log(microsecs);
}

function countToOneBillion(){
    var i = 0;
    while(i < 1000000000){
        i++;
    }
}

main();
```
  
##Full example

```js
var NanoTimer = require('nanotimer');

var count = 10;


function main(){
    var timer = new NanoTimer();
    
    timer.setInterval(countDown, '', '1s');
    timer.setTimeout(liftOff, [timer], '10s');
    


}

function countDown(){
    console.log('T - ' + count);
    count--;
}

function liftOff(timer){
    timer.clearInterval();
    console.log('And we have liftoff!');
}

main();
```

### In the above example, the interval can also be cleared another way rather than having to pass in the timer object to the liftOff task.
Instead, it can be done by specifying a callback to setTimeout, since the timer object will exist in that scope.  Like so:

```js
timer.setTimeout(liftOff, '', '10s', function(){
    timer.clearInterval();
});
```


## .setTimeout(task, args, timeout, [callback])
* Calls function 'task' with argument(s) 'args' after specified amount of time, 'timeout'.
* timeout, specified as a number plus a letter concatenated into a string. ex - '200u', '150n', '35m', '10s'.
* callback is optional.  If it is specified, it is called when setTimeout runs it's assigned task, and it is sent a parameter that
tells the actuam amount of time that passed before the specifed task was run, in nanoseconds.

```js
console.log("It's gonna be legen-wait for it...");

timerA.setTimeout(dary, '', '2s');

function dary(){
    console.log("dary!!");
}
```

## .setInterval(task, args, interval, [callback])
* Repeatedly calls function 'task' with arguments 'args' after every interval amount of time. If interval is specified as 0, it will run as fast as possible!
* This function is self correcting, error does not propagate through each cycle, as described above.
* interval, specified as a number plus a letter concatenated into a string. ex - '200u', '150n', '35m', '10s'.
* callback is optional, and is only called once the interval is cleared.

```js
timerA.setInterval(task, '100m', function(err) {
    if(err) {
        //error
    }
});
```

## .time(task, args, format, [callback])
* Returns the amount of time taken to run function 'task', called with arguments 'args'.
* format specifies the units time will be returned in. Options are 's' for seconds, 'm' for milliseconds, 'u' for microseconds, 
and 'n' for nanoseconds. if no format is specified, returns the default array of [s, n] where s is seconds and n is nanoseconds.
* callback is optional

### Synchronous Example:
```js

var runtimeSeconds = timerA.time(doMath, 'u');

function doMath(){
    //do math
}

```

### Asynchronous Use: (yes, it can time asynchronous stuff! here's how)

In order to time something asynchronous, it's not surprise that the timer object must somehow be notified whenever that asynchronous task finishes.
To do that, you must have your function accept a callback, and manually call the callback (to the timer function) inside of the asynchronous task's
callback. It's essentially a chain of callbacks, which is probably already familiar to you. Here's an example that times how long it takes to read a file. 
```js
var NanoTimer = require('nanotimer');
var fs = require('fs');

var timer = new NanoTimer();


timer.time(loadFile, '', 'u', function(time){
    console.log("It took " + time + " microseconds to read that file!");
});

function loadFile(callback){
    fs.readFile('testReadFile.js', function(err, data){
        if(err) throw err;
        console.log(data);
        
        callback();
    });
}
```

Once again, just two changes from normal.  First, your function to be timed, in this case 'loadFile' (which is just a proxy function to perfom fs.readFile) must
accept a callback parameter.  Second, in the callback of whatever asynchronous task is being performed inside the proxy, the callback passed in must be called after everything
is finished.  Internally, an unnamed function is presented as the callback to whatever the task is.  That callback that you call manually immediately takes the 2nd reference time,
and then calls the callback visible to it inside the timer, the callback function you specified to timer.time.

## .clearInterval()
* Clears current running interval
```js
timer.clearInterval();
```

#Logging
* Added preliminary logging feature.  If a timer is created by passing in 'log', it will enable verbose logging from the timer, so you can
figure out the real amount of time being taken for setTimeout or setInterval
* Currently only works on setInterval, and displays the 'cycle time', or real interval time to demonstrate how error does not propagate.  This will 
be further expanded on.

# Tests

* Test suite used is mocha.
* Tests also require **should**
* In order for the test to perform properly, the timeout must be altered.
* I prefer running tests with `mocha -R spec -t 10000`

![](https://raw.github.com/Krb686/nanotimer/master/test/nanotimer_0_2_6_test_partial.png "Test Results")

# Performance

-Soon to be added








