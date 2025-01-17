# win32-api
FFI Definitions of Windows win32 api for [node-ffi](https://github.com/node-ffi/node-ffi)

[![Version](https://img.shields.io/npm/v/win32-api.svg)](https://www.npmjs.com/package/win32-api)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Available platform](https://img.shields.io/badge/platform-win32-blue.svg)
[![Build status](https://ci.appveyor.com/api/projects/status/nrivtykm5uf84fbl/branch/master?svg=true)](https://ci.appveyor.com/project/waitingsong/node-win32-api/branch/master)
[![Coverage Status](https://coveralls.io/repos/github/waitingsong/node-win32-api/badge.svg)](https://coveralls.io/github/waitingsong/node-win32-api)
[![Conventional Commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-yellow.svg)](https://conventionalcommits.org)



## What can I do with this?
Calling win32 native functions come from user32.dll, kernel32.dll, comctl32.dll by Node.js via [node-ffi](https://github.com/node-ffi/node-ffi) or [node-ffi-napi](https://github.com/node-ffi-napi/node-ffi-napi)

## Installing
```powershell
npm install win32-api
```

## Usage

### Find window and set window title
```ts
// **Find calc's hWnd, need running a calculator program manually at first**

/**
 * expose module:
 * C, Comctl32 for Comctl32 from lib/comctl32/api
 * K, Kernel32 for kernel32 from lib/kernel32/api
 * U, User32 for user32 from lib/user32/api
 */
import { K, U } from 'win32-api'
import * as ref from 'ref-napi'

const knl32 = K.load()
const user32 = U.load()  // load all apis defined in lib/{dll}/api from user32.dll
// const user32 = U.load(['FindWindowExW'])  // load only one api defined in lib/{dll}/api from user32.dll

const title = 'Calculator\0'    // null-terminated string
// const title = '计算器\0'    // string in chinese

const lpszWindow = Buffer.from(title, 'ucs2')
const hWnd = user32.FindWindowExW(null, null, null, lpszWindow)

if (hWnd && ! hWnd.isNull()) {
  // Caution: outputing hWnd will cuase exception in the following process, even next script!
  // NOT do below in the production code!
  // console.log('buf: ', hWnd); // avoid this
  console.log('buf: ', ref.address(hWnd)) // this is ok

  // Change title of the Calculator
  const res = user32.SetWindowTextW(hWnd, Buffer.from('Node-Calculator\0', 'ucs2'))

  if ( ! res) {
    console.log('SetWindowTextW failed')
  }
  else {
    console.log('window title changed')
  }
}
```

### [Ref](https://www.npmjs.com/package/ref-napi)
```ts
import { U } from 'win32-api'
import * as ref from 'ref-napi'

// so we can all agree that a buffer with the int value written
// to it could be represented as an "int *"
const buf  = Buffer.alloc(4)
buf.writeInt32LE(12345, 0)

const hex = ref.hexAddress(buf)
console.log(typeof hex)
console.log(hex)  // ← '7FA89D006FD8'

buf.type = ref.types.int  // @ts-ignore

// now we can dereference to get the "meaningful" value
console.log(ref.deref(buf))  // ← 12345
```

```ts
// usage of types and windef:
import { K, FModel as FM, DTypes as W } from 'win32-api'
import * as ref from 'ref-napi'

const knl32 = K.load()

const buf = <FM.Buffer> Buffer.alloc(4)   // ← here the types
buf.writeInt32LE(12345, 0)

// const hInstance =<FM.Buffer> Buffer.alloc(process.arch === 'x64' ? 8 : 4)
const hInstance = <FM.Buffer> ref.alloc(W.HINSTANCE)    // W.HINSTANCE is 'int64*' under x64, 'int32*' under ia32
knl32.GetModuleHandleExW(0, null, hInstance)
```

### [Struct](https://www.npmjs.com/package/ref-struct)
```ts
// struct usage with ref-struct
import * as Struct from 'ref-struct'
import { DModel as M, DStruct as DS } from 'win32-api'

// https://msdn.microsoft.com/en-us/library/windows/desktop/dd162805(v=vs.85).aspx
const point: M.POINT_Struct = new Struct(DS.POINT)()
point.x = 100
point.y = 200
console.log(point)

// struct usage with ref-struct-di
import * as ref from 'ref-napi'
import * as StructDi from 'ref-struct-di'
import { DModel as M, DStruct as DS } from 'win32-api'

const Struct = StructDi(ref)
const point: M.POINT_Struct = new Struct(DS.POINT)()
point.x = 100
point.y = 200
console.log(point)
```

### Async Find window and set window title
```ts
// **Find calc's hWnd, need running a calculator program manually at first**

import { U } from 'win32-api'
import * as ref from 'ref-napi'


const u32 = U.load(['FindWindowExW', 'SetWindowTextW'])
const lpszClass = Buffer.from('CalcFrame\0', 'ucs2')

u32.FindWindowExW.async(null, null, lpszClass, null, (err, hWnd) => {
  if (err) {
    throw err
  }

  if (hWnd && !ref.isNull(hWnd) && ref.address(hWnd)) {
    const title = 'Node-Calculator'
    // Change title of the Calculator
    u32.SetWindowTextW.async(hWnd, Buffer.from(title + '\0', 'ucs2'), err2 => {
      if (err2) {
        throw err2
      }

      const buf = Buffer.alloc(title.length * 2)
      u32.GetWindowTextW.async(hWnd, buf, buf.byteLength, err3 => {
        if (err3) {
          throw err3
        }

        const str = buf.toString('ucs2').replace(/\0+$/, '')
        if (str !== title) {
          throw new Error(`title should be changed to ${title}, bug got ${str}`)
        }
      })
    })
  }
  else {
    throw new Error('FindWindowExW() failed')
  }
})
```


## Demo
- [create_window](https://github.com/waitingsong/node-win32-api/blob/master/demo/create_window.ts)
- [More](https://github.com/waitingsong/node-win32-api/blob/master/test)


## Dependencies Troubleshooting
- If installation of node-gyp fails:
Check out [node-gyp](https://github.com/nodejs/node-gyp) and [windows-build-tools](https://github.com/felixrieseberg/windows-build-tools)

## Relevant
- [Windows Api documentation](https://msdn.microsoft.com/en-us/library/windows/desktop/ff468919%28v=vs.85%29.aspx)
- [Windows Data Types](https://msdn.microsoft.com/en-us/library/windows/desktop/aa383751#DWORD)
- [System Error Codes](https://msdn.microsoft.com/en-us/library/windows/desktop/ms681381%28v=vs.85%29.aspx)
- [FFI doc](https://github.com/node-ffi/node-ffi/wiki/Node-FFI-Tutorial)
- [ref doc](https://tootallnate.github.io/ref/)
- [ref-struct](https://github.com/TooTallNate/ref-struct)


## License
[MIT](LICENSE)


### Languages
- [English](README.md)
- [中文](README.zh-CN.md)
