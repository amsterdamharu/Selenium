# Selenium with nodejs

Here is how you could set up end to end testing including comparing pages (what it looked like before and after).

First you need to set up nodejs for windows and make sure npm and node works in your command (set the path to the nodejs binaries).

Then; if you want to compare screen shots; install graphics magic for Windows: ftp://ftp.graphicsmagick.org/pub/GraphicsMagick/windows/GraphicsMagick-1.3.21-Q16-win32-dll.exe

Then the selenium driver for Internet Explorer (this will allow selenium to control Internet Explorer):  https://www.npmjs.com/package/selenium-webdriver has drivers for Edge and Chrome as well (for Internet Explorer use the 32 bit one because the 64 bit has some bugs but that may be solved by now)

Copy the webdriver executable to somewhere in your path (C:\windows\system32 for example)

Open Internet explorer and go to �Internet options� => security => for each zone either check or uncheck �Enable Protected Mode�. If they are not the same the driver will complain about it and exit your program.

Make sure zoom level in Internet explorer is on 100%. If it�s not the driver will complain and your program will exit.

Make a directory to hold your test code (for example E:\seleniumTests)
Make a directory to hold your images: (for example E:\seleniumTests\images)
Make a directory to hold you the differences in images: (for example E:\seleniumTests\images\diff)

Open a command prompt and execute the following commands in the directory you created:

npm install selenium-webdriver colors gm

This will install selenium-webdriver, colors and graphics magic:
https://www.npmjs.com/package/selenium-webdriver
https://github.com/Marak/colors.js
https://github.com/aheckmann/gm

Open E:\seleniumTest\node_modules\gm\lib\compare.js in an editor and comment out line 36 or 37:
//        options.highlightColor = '"' + options.highlightColor + '"';

Reference to the bug:
https://github.com/aheckmann/gm/issues/480

The following code will compare localhost test site with staging. You can change the tolerance to a lower number to make it check for differences in a stricter manner. If images (screenshots) are regarded as the same the diff image will be removed you can find the differences in: E:\seleniumTest\images\diff (or change the code to use the path that you created for the test).


The following code is messy and needs to be cleaned up. Testing code should never access webdriver logic and instead rely on functions such as openHomePage, logIn, clickSortByDate ...

        var webdriver = require('selenium-webdriver')
            , By = webdriver.By
            , until = webdriver.until
            , ie = require('selenium-webdriver/ie')
            , gm = require('gm')//.subClass({imageMagick: true});
            , fs = require('fs')
            , colors = require('colors')
            , driver = new webdriver.Builder()
                .forBrowser('ie')
                .setIeOptions(new ie.Options())
                .build();
        function compare(file1Path,file2Path,diffFilePath,tolerance){
            var p = new Promise(
                function(resolve,reject){
                    gm.compare(
                        file1Path, 
                        file2Path,
                        {
                            file: diffFilePath,
                            highlightColor: 'yellow',
                            highlightStyle:'Assign',
                            tolerance: tolerance
                        },
                        function (err, isEqual, equality, raw, path1, path2) {
                            if (err) {
                                reject(err);
                                return;
                            }
                            // if the images were considered equal, `isEqual` will be true, otherwise, false.
                            console.log('The images were equal: %s', isEqual);
                            // to see the total equality returned by graphicsmagick we can inspect the `equality` argument.
                            console.log('Actual equality: %d', equality);
                            if(!isEqual){
                                console.error(("----- Please inspect:"+diffFilePath).red);
                                resolve();
                            }else{
                                fs.unlink(diffFilePath,(err)=>{
                                    if(err){reject(err);return;}
                                    resolve();
                                });
                            }
                    });
                }
            );
            return p;
        }
        function writeScreenshot(data, name) {
        name = name || 'ss.png';
        //@todo: use path seperator instead and store the image location in a constant
        var screenshotPath = 'E:\\seleniumTest\\images\\';
        var p = new Promise(
            function(resolve,reject){
                fs.writeFile(screenshotPath + name, data, 'base64',(err)=>{
                    if(err){
                        reject(err);return;
                    }
                    resolve();
                });          
            }
        );
        return p;
        };
        function wait(ms,data){
            ms=ms||5000;
            var p = new Promise((resolve,reject)=>{
                setTimeout(()=>{resolve(data);},ms);
            });
            return p;
        }

        //compare staging with test site
        driver.get('http://localhost:9999/somePath')
        .then(function(){
        return wait(5000);//buggy implementation: do not use in real tests
        })
        .then(function(){
        return driver.takeScreenshot();
        })
        .then(function(data) {
        return writeScreenshot(data, 'out1.png');
        })

        driver.get('http://localhost/somePath')
        .then(function(){
        return wait(5000);//buggy implementation: do not use in real tests
        })
        .then(function(){
        return driver.takeScreenshot();
        })
        .then(function(data) {
        return writeScreenshot(data, 'out2.png');
        })

        .then(function(data) {
        return compare(
            'E:\\seleniumTest\\images\\out1.png', 
            'E:\\seleniumTest\\images\\out2.png',
            'E:\\seleniumTest\\images\\diff\\out1.png',
            0.001//tollerance for difference, the higher the more difference it allows
        );
        })
        .then(function(data) {
        driver.quit();
        })
        .then(null,function(fail){
        console.log("Failed:",fail);
        driver.quit();
        });


