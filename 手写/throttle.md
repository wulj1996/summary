function throttle(func, time ) {
let preTime = 0;
return function(...args) {
const that = this;
const curTime = Date.now();
if (curTime - preTime >= time) {
preTime = curTime;
func.apply(that, args);
}
}
}

function f() {
console.log(123);
}

const ff = throttle(f, 2000);

ff();

setInterval(() => {
ff();
}, 1000);
