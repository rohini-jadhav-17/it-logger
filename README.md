
// import fs from 'fs/promises';
// import path from 'path';
// import epub from 'epub';

// async function convertEpubToHtml(epubPath, outputPath) {
//     try {
//         // Check if the EPUB file exists
//         await fs.access(epubPath);

//         const book = new epub(epubPath);

//         book.on('error', (err) => {
//             console.error('Error:', err);
//         });

//         book.on('end', async () => {
//             let htmlContent = '<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><title>EPUB to HTML</title></head><body>';
            
//             for (const id of book.flow) {
//                 const chapter = await new Promise((resolve, reject) => {
//                     book.getChapter(id.id, (err, text) => {
//                         if (err) {
//                             reject(err);
//                         } else {
//                             resolve(text);
//                         }
//                     });
//                 });

//                 htmlContent += `<div>${chapter}</div>`;
//             }
            
//             htmlContent += '</body></html>';
            
//             // Write the HTML content to a file
//             await fs.writeFile(outputPath, htmlContent);
//             console.log('EPUB converted to HTML successfully');
//         });

//         book.parse();
//     } catch (error) {
//         console.error('Error converting EPUB to HTML:', error);
//     }
// }

// // Convert an EPUB file to HTML
// const epubPath = path.resolve('sample1.epub'); // Replace with the absolute path to your EPUB file
// const outputPath = path.resolve('./output1.html'); // Replace with the desired output path for the HTML file

// convertEpubToHtml(epubPath, outputPath);

import { promises as fsPromises, createReadStream, createWriteStream, constants } from 'fs';
import path from 'path';
import epub from 'epub';
import unzipper from 'unzipper';

async function extractCssFromEpub(epubPath, outputDir) {
    try {
        // Create output directory if it doesn't exist
        await fsPromises.mkdir(outputDir, { recursive: true });

        // Extract CSS files from EPUB
        await createReadStream(epubPath)
            .pipe(unzipper.Parse())
            .on('entry', async (entry) => {
                const fileName = entry.path;
                const fileType = entry.type; // 'Directory' or 'File'

                // Check if the entry is a CSS file
                if (fileType === 'File' && fileName.toLowerCase().endsWith('.css')) {
                    // Extract the CSS file
                    const outputPath = path.join(outputDir, path.basename(fileName));
                    await new Promise((resolve, reject) => {
                        entry.pipe(createWriteStream(outputPath))
                            .on('finish', resolve)
                            .on('error', reject);
                    });
                    console.log(`Extracted CSS file: ${outputPath}`);
                } else {
                    // Consume entry stream to move to the next entry
                    entry.autodrain();
                }
            })
            .promise();

        console.log('CSS files extracted successfully');
    } catch (error) {
        console.error('Error extracting CSS from EPUB:', error);
    }
}

async function convertEpubToHtml(epubPath, outputDir, cssDir) {
    try {
        // Check if the EPUB file exists
        await fsPromises.access(epubPath, constants.F_OK);

        const book = new epub(epubPath);

        book.on('error', (err) => {
            console.error('Error:', err);
        });

        book.on('end', async () => {
            // Extract CSS files from EPUB
            await extractCssFromEpub(epubPath, cssDir);

            // Generate HTML content with links to CSS files
            let htmlContent = '<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><title>EPUB to HTML</title>';

            // Add link to each CSS file in the CSS directory
            const cssFiles = await fsPromises.readdir(cssDir);
            cssFiles.forEach(async (cssFile) => {
                htmlContent += `<link rel="stylesheet" href="${path.join(cssDir, cssFile)}">`;
            });

            htmlContent += '</head><body>';

            // Extract and append HTML content
            for (const id of book.flow) {
                const chapter = await new Promise((resolve, reject) => {
                    book.getChapter(id.id, (err, text) => {
                        if (err) {
                            reject(err);
                        } else {
                            resolve(text);
                        }
                    });
                });

                htmlContent += `<div>${chapter}</div>`;
            }

            htmlContent += '</body></html>';

            // Write the HTML content to a file
            const outputHtmlPath = path.join(outputDir, 'output.html');
            await fsPromises.writeFile(outputHtmlPath, htmlContent);
            console.log('EPUB converted to HTML successfully');
        });

        book.parse();
    } catch (error) {
        console.error('Error converting EPUB to HTML:', error);
    }
}


// Example usage:
const epubPath = 'book.epub'; // Replace with the path to your EPUB file
const outputDir = './output'; // Replace with the desired output directory for HTML file
const cssDir = './output/css'; // Replace with the desired output directory for CSS files
convertEpubToHtml(epubPath, outputDir, cssDir);
