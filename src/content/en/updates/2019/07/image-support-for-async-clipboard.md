project_path: /web/_project.yaml
book_path: /web/updates/_book.yaml
description: Starting in Chrome 76, the Asynchronous Clipboard API now handles some images, in addition to text.

{# wf_published_on: 2019-07-03 #}
{# wf_updated_on: 2019-09-06 #}
{# wf_featured_image: /web/updates/images/generic/photo.png #}
{# wf_tags: capabilities,chrome76,cutandcopy,execcommand,input,clipboard #}
{# wf_featured_snippet: Chrome 76 adds expands the functionality of the Async Clipboard API to add support for png images. Copying and pasting images to the clipboard has never been easier. #}
{# wf_blink_components: Blink>DataTransfer #}

# Image Support for the Async Clipboard API {: .page-title }

{% include "web/_shared/contributors/thomassteiner.html" %}

In Chrome 66, we shipped the [text portion](/web/updates/2018/03/clipboardapi)
of the [Asynchronous Clipboard
API](https://w3c.github.io/clipboard-apis/#async-clipboard-api). Now in Chrome
76, adding support for images makes it easy to programmatically copy and paste
png images.

<aside class="caution">
  <b>Note:</b>
  At the time of writing, only png files are supported.
  Support for other images and file formats will be added in the future.
</aside>

Before we dive in, let’s take a brief look at how the Asynchronous Clipboard
API works. If you remember the details, skip ahead to the
[image section](#images).

## Recap of the Asynchronous Clipboard API {: #recap }

Before describing image support, I want to review how the Asynchronous Clipboard
API works. Feel free to [skip ahead](#images) if you're already comfortable
using the API.

### Copy: writing text to the clipboard {: #copy-text }

To copy text to the clipboard, call `navigator.clipboard.writeText()`.
Since this API is asynchronous, the `writeText()` function returns a promise
that resolves or rejects depending on whether the passed text is
copied successfully. For example:

```js
async function copyPageURL() {
  try {
    await navigator.clipboard.writeText(location.href);
    console.log('Page URL copied to clipboard');
  } catch (err) {
    console.error('Failed to copy: ', err);
  }
}
```

### Paste: reading text from the clipboard {: #reading-text }

Much like copy, text can be read from the clipboard by calling
`navigator.clipboard.readText()` and waiting for the returned promise to
resolve with the text:

```js
async function getClipboardText() {
  try {
    const text = await navigator.clipboard.readText();
    console.log('Clipboard contents: ', text);
  } catch (err) {
    console.error('Failed to read clipboard contents: ', err);
  }
}
```

### Handling paste events

Paste events can be handled by listening for the (surprise) `paste` event.
Note that you need to call `preventDefault()` in order to modify the to-be-pasted data,
like for example, convert it to uppercase before pasting.
It works nicely with the new asynchronous methods for reading clipboard text:

```js
document.addEventListener('paste', async (e) => {
  e.preventDefault();
  try {
    const text = await navigator.clipboard.readText();
    text = text.toUpperCase();
    console.log('Pasted UPPERCASE text: ', text);
  } catch (err) {
    console.error('Failed to read clipboard contents: ', err);
  }
});
```

### Security and permissions {: #security-permission }

The `navigator.clipboard` property is only supported for pages served over HTTPS,
and to help prevent abuse, clipboard access is only allowed when a page is
the active tab. Pages in active tabs can write to the clipboard without
requesting permission, but reading from the clipboard always requires
permission.

When the Asynchronous Clipboard API was introduced,
two new permissions for copy and paste were added to the
 [Permissions API](/web/updates/2015/04/permissions-api-for-the-web):

* The `clipboard-write` permission is granted automatically to pages when they
  are in the active tab.
* The `clipboard-read` permission must be requested, which you can do by
  trying to read data from the clipboard.

```js
const queryOpts = { name: 'clipboard-read' };
const permissionStatus = await navigator.permissions.query(queryOpts);
// Will be 'granted', 'denied' or 'prompt':
console.log(permissionStatus.state);

// Listen for changes to the permission state
permissionStatus.onchange = () => {
  console.log(permissionStatus.state);
};
```

## Images in the Asynchronous Clipboard API {: #images }

### Copy: writing an image to the clipboard {: #copy-image }

The new `navigator.clipboard.write()` method can be used for copying images
to the clipboard. Like [`writeText()`](#copy-text), it is asynchronous, and
Promise-based. Actually, `writeText()` is just a convenience method for the
generic `write()` method.

To write an image to the clipboard, you need the image as a
[`Blob`][blob]. One way to do
this is by requesting the image from an server by calling `fetch()` (or
`XMLHTTPReuest()`). The `response` object returned by `fetch()` has a [`blob()`
method][blob-method] and
`XMLHTTPRequest()` let's you set `"blob"` as the [`responseType`][blog-response-type].

Calling the server may not be desireable or possible for a variety of reasons.
Fortunately, you can also write the image to a canvas, then call
[`HTMLCanvasElement.toBlog()`][to-blob].

[blob]: https://developer.mozilla.org/en-US/docs/Web/API/blob
[blob-method]: https://developer.mozilla.org/en-US/docs/Web/API/Body/blob
[to-blob]: https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toBlob
[blog-response-type]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/responseType#Value

Next, pass an array of `ClipboardItem` objects as a parameter to the `write()` method.
Currently you can only pass one image at a time, but we plan to add support for
multiple images in the future.

The `ClipboardItem` takes an object with the MIME type of the image as the key,
and the actual blob as the value. The sample code below shows a future-proof way
to do this by using the [`Object.defineProperty()`][object-define-prop] method.
The MIME used as the key is retrieved from `blob.type`. This approach ensures
that your code will be ready for future image types as well as other MIME types
that may be supported in the future.

[object-define-prop]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

```js
try {
  const imgURL = 'https://developers.google.com/web/updates/images/generic/file.png';
  const data = await fetch(imgURL);
  const blob = await data.blob();
  await navigator.clipboard.write([
    new ClipboardItem(Object.defineProperty({}, blob.type, {
      value: blob,
      enumerable: true
    }))
  ]);
  console.log('Image copied.');
} catch(e) {
  console.error(e, e.message);
}
```

### Paste: reading an image from the clipboard {: #paste-image }

The `navigator.clipboard.read()` method, which reads data from the clipboard, is
also asynchronous, and Promise-based.

To read an image from the clipboard, obtain a list of
`ClipboardItem` objects, then iterate over them. Since everything is asynchronous,
use the [`for ... of`][for-of] iterator, since it handles async/await code
nicely.

[for-of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of

Each `ClipboardItem` can hold its contents in different types, so you'll
need to iterate over the list of types, again using a `for ... of` loop.
For each type, call the `getType()` method with the current type as an argument
to obtain the corresponding image `Blob`. As before, this code is not tied
to images, and will work with other future file types.

```js
async function getClipboardContents() {
  try {
    const clipboardItems = await navigator.clipboard.read();
    for (const clipboardItem of clipboardItems) {
      try {
        for (const type of clipboardItem.types) {
          const blob = await clipboardItem.getType(type);
          console.log(URL.createObjectURL(blob));
        }
      } catch (e) {
        console.error(e, e.message);
      }
    }
  } catch (e) {
    console.error(e, e.message);
  }
}
```

### Custom paste handler {: #custom-paste-handler }

To dynamically handle paste events, listen for the `paste`
event, call [`preventDefault()`][prevent-default] to prevent the default behavior
in favor of your own logic, then use the code above
to read the contents from the clipboard, and handle it in whatever way your
app needs.

[prevent-default]: https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault

```js
document.addEventListener('paste', async (e) => {
  e.preventDefault();
  getClipboardContents();
});
```

### Custom copy handler {: #custom-copy-handler }

The [`copy` event][copy-event] includes a [`clipboardData`][clipboard-data]
property with the items already in the right format, eliminating the need to
manually create a blob. As before, don't forget to call `preventDefault()`.

[copy-event]: https://developer.mozilla.org/en-US/docs/Web/API/Document/copy_event
[clipboard-data]: https://developer.mozilla.org/en-US/docs/Web/API/ClipboardEvent/clipboardData

```js
document.addEventListener('copy', async (e) => {
  e.preventDefault();
  try {
    for (const item of e.clipboardData.items) {
      await navigator.clipboard.write([
        new ClipboardItem(Object.defineProperty({}, item.type, {
          value: item,
          enumerable: true
        }))
      ]);
    }
    console.log('Image copied.');
  } catch(e) {
    console.error(e, e.message);
  }
});
```

### Demo

<iframe style="border: solid 1px;" width="100%" height="500"
  src="https://jsfiddle.net/0794oysr/2/embedded/result,js,html,css"
  allowfullscreen="allowfullscreen" frameborder="0">
</iframe>

### Security

Opening up the Asynchronous Clipboard API for images comes with certain
[risks](https://w3c.github.io/clipboard-apis/#security) that need to be carefully evaluated.
One new challenge are so-called [image compression bombs](https://bomb.codes/bombs#images),
that is, image files that appear to be innocent, but—once decompressed—turn out to be huge.
Even more serious than large images are specifically crafted malicious images
that are designed to exploit known vulnerabilities in the native operating system.
This is why we can’t just copy the image directly to the native clipboard,
and why in Chrome we require that the image be transcoded.

The [specification](https://w3c.github.io/clipboard-apis/#image-transcode)
therefore also explicitly mentions transcoding as a mitigation method:
*“To prevent malicious image data from being placed on the clipboard, the image data may be
transcoded to produce a safe version of the image.”*
There is ongoing discussion happening in the
[W3C Technical Architecture Group review](https://github.com/w3ctag/design-reviews/issues/350)
on whether, and how the transcoding details should be specified.

## Next Steps

We are actively working on expanding the Asynchronous Clipboard API to add
support a larger number of data types. Due to the potential risks we are
treading carefully. To stay up to date on Chrome's progress, you can star the
[bug][cr-bug] to be notified about changes.

For now, image support can be used in  Chrome 76 or later.

Happy copying and pasting!

[cr-bug]: https://bugs.chromium.org/p/chromium/issues/detail?id=897289

## Related links

* [Explainer](https://github.com/w3c/clipboard-apis/blob/master/explainer.adoc)
* [Raw Clipboard Access Design Doc](https://docs.google.com/document/d/1XDOtTv8DtwTi4GaszwRFIJCOuzAEA4g9Tk0HrasQAdE/edit?usp=sharing)
* [Chromium bug](https://crbug.com/150835)
* [Chrome Platform Status entry](https://www.chromestatus.com/features/5074658793619456)
* [Technical Architecture Group (TAG) Review](https://github.com/w3ctag/design-reviews/issues/350)

## Acknowledgements

The Asynchronous Clipboard API was implemented by
[Darwin Huang](https://www.linkedin.com/in/darwinhuang/)
and [Gary Kačmarčík](https://www.linkedin.com/in/garykac/).
Darwin also provided the [demo](https://jsfiddle.net/0794oysr/2/).
My introduction of this article is inspired by
[Jason Miller](https://twitter.com/_developit?lang=en)’s
original [text](/web/updates/2018/03/clipboardapi).
Thanks to [Kyarik](https://github.com/kyarik) and again Gary Kačmarčík for reviewing this article.

## Feedback {: #feedback .hide-from-toc }

{% include "web/_shared/helpful.html" %}

{% include "web/_shared/rss-widget-updates.html" %}
