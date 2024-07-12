# Custom matcher for testing PDFs
Here's a simple example of a custom matcher for [Playwright](https://playwright.dev/) that can be used for testing PDFs in a TypeScript test project. This custom matcher convert the PDF file to PNGs (one PNG per page in the pdf) using [pdf-to-png-converter](https://www.npmjs.com/package/pdf-to-png-converter) and then uses [Pixelmatch](https://www.npmjs.com/package/pixelmatch) for checking the pixel diffs. If there's pixel diffs the matcher will attach the 'expected', 'actual' and 'diff' images to the test report.

```TypeScript
import { test, expect as baseExpect } from '@playwright/test';
import Pixelmatch from 'pixelmatch';
import { PNG } from 'pngjs';
import { pdfToPng } from 'pdf-to-png-converter';

export const expect = baseExpect.extend({
  async toMatchPdf(actual: Buffer, expected: Buffer) {
    // check if the buffers are PDF by checking the first 4 bytes if they match the PDF magic number (see https://en.wikipedia.org/wiki/Magic_number_(programming))
    if (actual.subarray(0, 4).toString('hex') !== '25504446' || expected.subarray(0, 4).toString('hex') !== '25504446') {
      return {
        message: () => 'Some of the inputs are not PDFs',
        pass: false,
      };
    }
    // convert the PDFs to PNGs
    const [actualPngPics, expectedPngPics] = await Promise.all([pdfToPng(actual), pdfToPng(expected)]);
    if (actualPngPics.length !== expectedPngPics.length) {
      return {
        message: () => `Different number of pages in the PDFs. Actual: ${actualPngPics.length}, Expected: ${expectedPngPics.length}`,
        pass: false,
      };
    }
    let pass = true;
    let msg = 'Pixel diff found on the following pages: ';
    for (let i = 0; i < actualPngPics.length; i++) {
      // encode the PNG buffers to PNG objects so Pixelmatch can be used (see https://github.com/mapbox/pixelmatch/issues/66)
      const actual = PNG.sync.read(actualPngPics[i].content);
      const expected = PNG.sync.read(expectedPngPics[i].content);

      const {width, height} = actual;
      const diffPng = new PNG({width, height});
      const pixelDiffNbr = Pixelmatch(actual.data, expected.data, diffPng.data, width, height);
      if (pixelDiffNbr > 0) {
        pass &&= false;
        msg += `${i + 1}, `;

        await Promise.all([
          test.info().attach(`Expected PDF page ${i + 1}`, {
            body: PNG.sync.write(expected), // decode the PNG object to a buffer
            contentType: 'image/png',
          }),
          test.info().attach(`Actual PDF page ${i + 1}`, {
            body: PNG.sync.write(actual),
            contentType: 'image/png',
          }),
          test.info().attach(`Pixel diff on page ${i + 1}`, {
            body: PNG.sync.write(diffPng),
            contentType: 'image/png',
          }),
        ]);
      }
    }

    return {
      message: () => pass ? '' : msg,
      pass: pass,
    };
  },
});
```
and here is a sample snippet on how to use it:

```TypeScript
function readableToBuffer(readable): Promise<Buffer> {
  return new Promise((resolve, reject) => {
    const chunks = [];
    readable.on('data', chunk => chunks.push(chunk));
    readable.on('error', reject);
    readable.on('end', () => resolve(Buffer.concat(chunks)));
  });
}

test('testing PDF', async ({page}) => {

  const pathToPdf = "<path_to_golden_pdf_file>"
  const originalPdfBuffer = fs.readFileSync(pathToPdf);
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    downloadPdfButton.click(),
  ]);

  // create a buffer from Readable stream
  const downloadPdfBuffer = await readableToBuffer(await download.createReadStream());
  await expect(downloadPdfBuffer).toMatchPdf(originalPngBuffer);
});

```
