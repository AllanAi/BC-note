## 本地搭建 remix

1. 下载官方包：https://github.com/ethereum/remix-project/releases remix-***.zip
2. 修改 main.js

删除提示：

```
The Remix IDE has moved to http://remix.ethereum.org. This instance of Remix you are visiting WILL NOT BE UPDATED. Please make a backup of your contracts and start using http://remix.ethereum.org
```

下载所有版本的solc.js到当前文件夹（自动跳过已下载的）
`node download-solc.js`

```js
const http = require('http');
const fs = require('fs');
const crypto = require('crypto')

const solc_info = {
    host: 'solc-bin.ethereum.org',
    wasm: 'wasm',
    bin: 'bin',
    list: 'list.json'
}
const wasmlist = `${solc_info.wasm}/${solc_info.list}`
const binlist = `${solc_info.bin}/${solc_info.list}`
let wasms = new Set()

const callback = function(listpath, type) {
    return function(response) {
        let str = '';

        //another chunk of data has been received, so append it to `str`
        response.on('data', function (chunk) {
            str += chunk;
        });

        //the whole response has been received, so we just print it out here
        response.on('end', function () {
            //console.log(str);
            fs.writeFile(listpath, str, err => {
                if (err) console.log(err)
            })
            const builds = JSON.parse(str).builds
            for (let i in builds) {
                let filename = builds[i].path
                if (type === solc_info.wasm) {
                    wasms.add(filename)
                }
                if (filename.indexOf('nightly') < 0 && (type === solc_info.wasm || !wasms.has(filename))) {
                    let filepath = type+"/"+filename
                    let h = builds[i].sha256
                    download(`http://${solc_info.host}/${type}/${filename}`, filepath, checkHashCallback(filepath, h))
                }
            }
            if (type === solc_info.wasm) {
                http.request({
                    host: solc_info.host,
                    path: `/${binlist}`
                }, callback(binlist, solc_info.bin)).end();
            }
        });
    }
}

const createDir = function(dir) {
    if (!fs.existsSync(dir)){
        fs.mkdirSync(dir);
    }
}

createDir(solc_info.wasm)
createDir(solc_info.bin)

// download wasm solcs
http.request({
    host: solc_info.host,
    path: `/${wasmlist}`
}, callback(wasmlist, solc_info.wasm)).end();



// download bin/list.json
// http.get(`http://${solc_info.host}/${binlist}`, function(response) {
//     if (response.statusCode === 200) {
//         response.pipe(fs.createWriteStream(binlist));
//     }
// });

const checkHashCallback = function(dest, hash){
    return function() {
        const sha256 = crypto.createHash('sha256')
        const file = fs.createReadStream(dest);
        file.on('readable', () => {
            const data = file.read();
            if (data)
                sha256.update(data);
            else {
                const _hash = '0x'+sha256.digest('hex')
                if ( _hash !== hash) {
                    console.error(`${dest} failed: ${hash} ${_hash}`);
                    fs.unlinkSync(dest)
                } else {
                    console.log(dest, 'successes')
                }
            }
        });
    }
}

const download = function(url, dest, cb) {
    if (fs.existsSync(dest)) { // skip files already download
        cb()
    } else {
        console.log(url, 'downloading...')
        const file = fs.createWriteStream(dest);
        const request = http.get(url, function(response) {
            response.pipe(file);
            file.on('finish', function() {
                file.close(cb);  // close() is async, call cb after close completes.
            });
        }).on('error', function(err) { // Handle errors
            fs.unlink(dest, () => {}); // Delete the file async. (But we don't check the result)
            if (cb) cb(err.message);
        });
    }
};
```

下载后，复制wasm和bin文件夹到remix目录

可以增加注释，查看所有使用的solcjs

```js
if (_this5._shouldBeAdded(option.innerText)) {
   if ('builtin' !== build.path) console.log('build', (0, _compilerUtils.urlFromVersion)(build.path))
  _this5._view.versionSelector.appendChild(option);
}
```

替换 `https://solc-bin.ethereum.org` 为 `http://'+window.location.host+'`

```
:%s/https:\/\/solc-bin.ethereum.org/http:\/\/'+window.location.host+'/g
```

（baseURLBin、baseURLWasm）

替换metadata.json(先下载到remix目录)

```
https://raw.githubusercontent.com/ethereum/remix-plugins-directory/master/build/metadata.json
http://'+window.location.host+'/metadata.json
```

注释包含 `https://platform.twitter.com/widgets.js` 的代码

```js
function _templateObject7() {
  //var data = _taggedTemplateLiteral(["\n      <div class=\"px-2 ", "\">\n        <a class=\"twitter-timeline\"\n          data-width=\"350\"\n          data-theme=\"", "\"\n          data-chrome=\"nofooter noheader transparent\"\n          data-tweet-limit=\"8\"\n          href=\"https://twitter.com/EthereumRemix\"\n        >\n        </a>\n        <script async src=\"https://platform.twitter.com/widgets.js\" charset=\"utf-8\"></script>\n      </div>\n    "]);
  var data = _taggedTemplateLiteral([""]);
  _templateObject7 = function _templateObject7() {
    return data;
  };

  return data;
}
function _templateObject2() {
  //var data = _taggedTemplateLiteral(["\n      <div class=\"px-2 ", "\">\n        <a class=\"twitter-timeline\"\n          data-width=\"350\"\n          data-theme=\"", "\"\n          data-chrome=\"nofooter noheader transparent\"\n          data-tweet-limit=\"8\"\n          href=\"https://twitter.com/EthereumRemix\"\n        >\n        </a>\n        <script async src=\"https://platform.twitter.com/widgets.js\" charset=\"utf-8\"></script>\n      </div>\n    "]);
  var data = _taggedTemplateLiteral([""]);
  _templateObject2 = function _templateObject2() {
    return data;
  };

  return data;
}
```

3.修改index.html
下载并替换
https://use.fontawesome.com/releases/v5.8.1/css/all.css ➡️ fontawesome.v5.8.1.all.css
https://kit.fontawesome.com/41dd021e94.js ➡️ fontawesome.41dd021e94.js