# unidbg 使用

前提环境: Java + Maven

使用IDEA克隆项目到本地: [zhkl0228/unidbg: Allows you to emulate an Android native library, and an experimental iOS emulation](https://github.com/zhkl0228/unidbg)

参考项目给出的样例: unidbg-android/src/test/java/com/anjuke/mobile/sign/SignUtil.java

搭建环境以及资源, so文件区分64位和32位, 传入apk文件无需补环境

[![](http://qiniu.yujing.icu/typora_img/image-20250314140306213.png)](http://qiniu.yujing.icu/typora_img/image-20250314140306213.png)

image-20250314140306213

初始化环境

```Java
package com.baidu.homework;/** * @author Tiam
 * @date 2025/1/8 10:39 * @description Bean
 */public class BaseUtil extends AbstractJni {    private final AndroidEmulator emulator;    private final VM vm;    private final Module module;    private final DvmClass cNativeHelper;    public static String pkgName = "com.baidu.homework";    // public static String apkPath = "zyb/作业帮14.17.0.apk";    public static String soPath = "unidbg-android/src/test/resources/zyb/libbaseutil.so";    public static String apkPath = "unidbg-android/src/test/resources/zyb/作业帮_14.18.0_APKPure.apk";    public BaseUtil() {        // 1. 创建 64位模拟器设备, so应该对应64位的        EmulatorBuilder<AndroidEmulator> builder = AndroidEmulatorBuilder.for64Bit()                // 指定进程名，推荐以安卓包名做进程名                .setProcessName(pkgName);        emulator = builder.build();        // 2. 获取 emulator 内存管理器        Memory memory = emulator.getMemory();        memory.setLibraryResolver(new AndroidResolver(23));        // 3. 指定apk文件, 创建虚拟机 new File(apkPath)        vm = emulator.createDalvikVM();        // vm.setDvmClassFactory(new ProxyClassFactory());        vm.setJni(this); // 绑定 JNI 接口, 后续设置补环境 !!!!        vm.setVerbose(true); // 是否打印日志        // emulator.getSyscallHandler().addIOResolver(this);        cNativeHelper = vm.resolveClass("com/zuoyebang/baseutil/NativeHelper");        // 4. 加载执行 so 文件        DalvikModule dm = vm.loadLibrary(new File(soPath), false);        dm.callJNI_OnLoad(emulator);        // 5. 获取本SO模块的句柄,后续需要用它        module = dm.getModule();        System.out.println("so在Unidbg虚拟内存中的基地址: " + module.base);    }    public static void main(String[] args) {        BaseUtil baseUtil = new BaseUtil();    }}
```

## 补环境

[![](http://qiniu.yujing.icu/typora_img/image-20250314141114089.png)](http://qiniu.yujing.icu/typora_img/image-20250314141114089.png)

image-20250314141114089

补环境大概需要补下面三个方法, 具体值可临时使用apk文件运行查看到

关键点: 值的类型对得上, 被校验的值符合

[![](http://qiniu.yujing.icu/typora_img/image-20250314155918180.png)](http://qiniu.yujing.icu/typora_img/image-20250314155918180.png)

image-20250314155918180

```Java
   @Override    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {        switch (signature) {            case "android/content/Context->getPackageName()Ljava/lang/String;":                return new StringObject(vm, "com.baidu.homework");            case "android/content/pm/Signature->toCharsString()Ljava/lang/String;":                return new StringObject(vm, "308201923081fca00302010202044d3c2820300d06092a864886f70d0101050500300d310b300906035504031302796b3020170d3131303132333133303734345a180f32303831303130353133303734345a300d310b300906035504031302796b30819f300d06092a864886f70d010101050003818d003081890281810095a1a931cc6bbc8899441e614f469104e2520a95ff90942ba177d336d98b1a5d6a637a0e95d1a3cc630537ecb1a5c708b5751d8f13bf8ba993b95748ed15b87c6dc22bf76e97f7ad68d86cad686752a48ce0cba009065a5f17650ab2301b9b871e3d0682712c0914a6b97df5b15ad15c14f080410b562973f830f31a31a75f970203010001300d06092a864886f70d0101050500038181002e5332040bde9448f53c63472c3a210da2101afe353538253072d643f089eb7eab68f0db2cedfb115bb73d2116db5fc0f516259f41ac0c04ee3b5e00710469d654b2d17a8330ad601e58f8d630afbc75420b9c55f62de033bcf02bdd9a1014d376576d048bebbe84a88826d9230527b5078bf08724cafb847ae64fa0e9aca40f");        }        return super.callObjectMethod(vm, dvmObject, signature, varArg);    }    @Override    public DvmObject<?> getObjectField(BaseVM vm, DvmObject<?> dvmObject, String signature) {        switch (signature) {            case "android/content/pm/PackageInfo->signatures:[Landroid/content/pm/Signature;":                // 只通过hashcode和toCharString()对比校验, 此处无所谓                byte[] sign = new byte[256];                // 在初始化代码中创建 Signature 对象                DvmClass signatureClass = vm.resolveClass("android/content/pm/Signature");                DvmObject<?> signature1 = signatureClass.newObject(sign); // 填入签名字节                // 构建 signatures 数组                DvmObject<?>[] signatures = {signature1};                return new ArrayObject(signatures);        }        return super.getObjectField(vm, dvmObject, signature);    }    @Override    public int callIntMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {        switch (signature) {            case "android/content/pm/Signature->hashCode()I":                int hash = Arrays.hashCode(signature.getBytes());                System.out.println("hash: " + hash);                return 0xd5fbc2fa;        }        return super.callIntMethod(vm, dvmObject, signature, varArg);    }
```

## 查签名

[![](http://qiniu.yujing.icu/typora_img/image-20250314151226274.png)](http://qiniu.yujing.icu/typora_img/image-20250314151226274.png)

image-20250314151226274

`char *__fastcall getAndroidSignatures(JNIEnv *env, jobject obj, jobject context)` 方法返回为apk原始签名数据的`hex`形式

[![](http://qiniu.yujing.icu/typora_img/image-20250317100633630.png)](http://qiniu.yujing.icu/typora_img/image-20250317100633630.png)

image-20250317100633630

`android/content/pm/Signature->toCharsString()Ljava/lang/String;` 补的值如下, `objSpamServer.cellPhone->androidSignatures`的值同样为此

```Plain
308201923081fca00302010202044d3c2820300d06092a864886f70d0101050500300d310b300906035504031302796b3020170d3131303132333133303734345a180f32303831303130353133303734345a300d310b300906035504031302796b30819f300d06092a864886f70d010101050003818d003081890281810095a1a931cc6bbc8899441e614f469104e2520a95ff90942ba177d336d98b1a5d6a637a0e95d1a3cc630537ecb1a5c708b5751d8f13bf8ba993b95748ed15b87c6dc22bf76e97f7ad68d86cad686752a48ce0cba009065a5f17650ab2301b9b871e3d0682712c0914a6b97df5b15ad15c14f080410b562973f830f31a31a75f970203010001300d06092a864886f70d0101050500038181002e5332040bde9448f53c63472c3a210da2101afe353538253072d643f089eb7eab68f0db2cedfb115bb73d2116db5fc0f516259f41ac0c04ee3b5e00710469d654b2d17a8330ad601e58f8d630afbc75420b9c55f62de033bcf02bdd9a1014d376576d048bebbe84a88826d9230527b5078bf08724cafb847ae64fa0e9aca40f
```

---

搜题酱

[LibChecker/LibChecker: An app to view libraries used in apps in your device.](https://github.com/LibChecker/LibChecker)

[![](http://qiniu.yujing.icu/typora_img/image-20250317164633972.png)](http://qiniu.yujing.icu/typora_img/image-20250317164633972.png)

image-20250317164633972

[![](http://qiniu.yujing.icu/typora_img/image-20250321155813891.png)](http://qiniu.yujing.icu/typora_img/image-20250321155813891.png)

image-20250321155813891

```Shell
keytool -printcert -jarfile 作业帮_14.18.0_APKPure.apk
"D:\ProgramDev\android-sdk\build-tools\apksigner.bat" verify -v --print-certs 作业帮_14.18.0_APKPure.apk
D:\Users\Tiam\scoop\apps\git\2.48.1\mingw64\bin\openssl.exe pkcs7 -in "D:\MuChenAI\Projects\zyb-apk-reverse\作业帮_14.18.0_APKPure\META-INF\ZHIDAO.RSA" -inform DER -print_certs -out cert.der
```

## 日志

通过两处地方控制日志

[![](http://qiniu.yujing.icu/typora_img/image-20250314161136338.png)](http://qiniu.yujing.icu/typora_img/image-20250314161136338.png)

v

## 调用JNI函数

通过**函数描述符(SymbolName)**调用 or 通过地址调用

[![](http://qiniu.yujing.icu/typora_img/image-20250314165408354.png)](http://qiniu.yujing.icu/typora_img/image-20250314165408354.png)

image-20250314165408354

或者通过偏移地址 `0x1060` 调用, 比较麻烦

```Java
    public void test() {        String cuid = "7AF43243C094AA9A44325CDA78FC5CB5|0";        Number number = module.callFunction(emulator, 0x1060,                vm.getJNIEnv(),                0,                vm.addLocalObject(vm.resolveClass("android/content/Context").newObject(null)),                vm.addLocalObject(new StringObject(vm, cuid))        );        System.out.println("result: " + vm.getObject(number.intValue()).getValue());    }
```

函数偏移地址还可以在IDA中查到

[![](http://qiniu.yujing.icu/typora_img/image-20250314181920484.png)](http://qiniu.yujing.icu/typora_img/image-20250314181920484.png)

image-20250314181920484

## Hook

为避免随机值的影响, hook随机值为固定值

```Java
public void hook_rand() {    IHookZz hookZz = HookZz.getInstance(emulator);    hookZz.enable_arm_arm64_b_branch();    hookZz.wrap(module.findSymbolByName("rand"), new WrapCallback<HookZzArm64RegisterContext>() {        @Override        public void preCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {        }        @Override        public void postCall(Emulator<?> emulator, HookZzArm64RegisterContext ctx, HookEntryInfo info) {            ctx.setXLong(0, 1L);        }    });}
```

固定时间戳

```Java
public void hook_time() {        HookZz instance = HookZz.getInstance(emulator);        instance.wrap(module.findSymbolByName("gettimeofday"), new WrapCallback<HookZzArm32RegisterContext>() {            UnidbgPointer tv = null;  // 初始化Pointer指针            @Override  // hook前            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {                tv = ctx.getPointerArg(0);  // 将指针赋值给tv            }            @Override // hook后            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {                if (tv != null) {                    byte[] before = tv.getByteArray(0, 12);                    Inspector.inspect(before, "gettimeofday tv");                }                System.out.println("====++++====");                // 固定时间                long currentTimeMillis = 1668083944037L;                long tv_sec = currentTimeMillis / 1000;                long tv_usec = (currentTimeMillis % 1000) * 1000;                System.out.println("=======");                System.out.println(currentTimeMillis);                System.out.println(tv_sec);                System.out.println(tv_usec);                // 创建TimeVal32时间对象，并传入指针                TimeVal32 TimeVal = new TimeVal32(tv);                TimeVal.tv_sec = (int) tv_sec;                TimeVal.tv_usec = (int) tv_usec;                TimeVal.pack();  // 替换            }        });    }
```

## 断点调试

[unidbg console debugger 调试技巧_unidbg debugger-CSDN博客](https://blog.csdn.net/linchaolong/article/details/142903088)

指令帮助

```YAML
c: continuen: step overbt: back tracest hex: search stackshw hex: search writable heapshr hex: search readable heapshx hex: search executable heapnb: break at next blocks|si: step intos[decimal]: execute specified amount instructions(bl): execute util BL mnemonic, low performancem(op) [size]: show memory, default size is 0x70, size may hex or decimalmx0-mx28, mfp, mip, msp [size]: show memory of specified registerm(address) [size]: show memory of specified address, address must start with 0xwx0-wx28, wfp, wip, wsp <value>: write specified registerwb(address), ws(address), wi(address), wl(address) <value>: write (byte, short, integer, long) memory of specified address, address must start with 0xwx(address) <hex>: write bytes to memory at specified address, address must start with 0xb(address): add temporarily breakpoint, address must start with 0x, can be module offsetb: add breakpoint of register PCr: remove breakpoint of register PCblr: add temporarily breakpoint of register LRp (assembly): patch assembly at PC addresswhere: show java stack tracetrace [begin end]: Set trace instructionstraceRead [begin end]: Set trace memory readtraceWrite [begin end]: Set trace memory writevm: view loaded modulesvbs: view breakpointsd|dis: show disassembled(0x): show disassemble at specify addressstop: stop emulationrun [arg]: run testgc: Run System.gc()threads: show thread listcc (size): convert asm from (0x120016c8) to (0x120016c8 + size) bytes to c function
```

[![](http://qiniu.yujing.icu/typora_img/image-20250314164423051.png)](http://qiniu.yujing.icu/typora_img/image-20250314164423051.png)

image-20250314164423051

断点`nativeInitBaseUtil`

[![](http://qiniu.yujing.icu/typora_img/image-20250314174628927.png)](http://qiniu.yujing.icu/typora_img/image-20250314174628927.png)

image-20250314174628927

[![](http://qiniu.yujing.icu/typora_img/image-20250314190657679.png)](http://qiniu.yujing.icu/typora_img/image-20250314190657679.png)

image-20250314190657679

机器码转汇编带代码

[Online ARM to HEX Converter](https://armconverter.com/)

## 查看变量值

[![](http://qiniu.yujing.icu/typora_img/image-20250315190322873.png)](http://qiniu.yujing.icu/typora_img/image-20250315190322873.png)

image-20250315190322873

```C
v6 = CRYStringCat("%s##%s##%s##%s", objSpamServer.prefix, output, v5, objSpamServer.cellPhone->androidDeviceID);v6 = 8&%d*#\#BBBBBBBBBB#\#0f3c509eef614432e414ce9d37f00c80#\#7AF43243C094AA9A44325CDA78FC5CB5|087位
objSpamServer.prefix:  8&%d*// 随机值(hook 固定了)output: BBBBBBBBBB
// CRYMd5(objSpamServer.cellPhone->androidSignatures)  签名值的MD5v5: 0f3c509eef614432e414ce9d37f00c80objSpamServer.cellPhone->androidDeviceID: 7AF43243C094AA9A44325CDA78FC5CB5|0
```

```C
// 输入: 8&%d*#\#BBBBBBBBBB#\#0f3c509eef614432e414ce9d37f00c80#\#7AF43243C094AA9A44325CDA78FC5CB5|0    魔改DES-ECB, 密钥 @fG2SuLA
   // 176位 8df33e23f867646595a8e3a2b3f01969ed0c5fb2ad95da17953beafc595ae2bab6935cb142a8a8611df71982a231b58649894b25227ef944c540757649fb9794454753c1bc4fce360b12311bfbd96335b0bfdc55faf3fc7d// 转位 352位, 0b010c0f070c0c04010f0e0602060a060a0901050c0704050c0d000f090809060b0703000f0a040d0b050a09050b0e080a090d0c0507030f090a050a0407050d060d0c09030a080d04020105010508060b080e0f090804010405080c0a0d0601090209010d020a040404070e090f02020a0300020a0e060e09020d0f0e0902090a020e020c0a0803030d0f020703060c0d000408080c0d080d0f090b0c060a0c000d0f0d030b0a0a050f0c0f030f0b0e
```

对比标准的`DES-ECB`实现结果并不一致

[![](http://qiniu.yujing.icu/typora_img/image-20250315192324388.png)](http://qiniu.yujing.icu/typora_img/image-20250315192324388.png)

image-20250315192324388

[![](http://qiniu.yujing.icu/typora_img/image-20250315192500939.png)](http://qiniu.yujing.icu/typora_img/image-20250315192500939.png)

image-20250315192500939

Python标准加密库: [Legrandin/pycryptodome: A self-contained cryptographic library for Python](https://github.com/Legrandin/pycryptodome)

选中左侧目录, Ctrl+F搜索函数名, 打开默认是汇编视图

[![](http://qiniu.yujing.icu/typora_img/image-20250315211524584.png)](http://qiniu.yujing.icu/typora_img/image-20250315211524584.png)

image-20250315211524584

快捷键`F5`反汇编到伪代码视图, 双击`DES_Encrypt`进入函数实现

[![](http://qiniu.yujing.icu/typora_img/image-20250315211658723.png)](http://qiniu.yujing.icu/typora_img/image-20250315211658723.png)

image-20250315211658723

再双击`DES_Encrypt`进入汇编

[![](http://qiniu.yujing.icu/typora_img/image-20250315211906013.png)](http://qiniu.yujing.icu/typora_img/image-20250315211906013.png)

image-20250315211906013

这里都是DES加密用到的相关常量, 右键可转为十进制, 方便对比

[![](http://qiniu.yujing.icu/typora_img/image-20250315212021494.png)](http://qiniu.yujing.icu/typora_img/image-20250315212021494.png)

image-20250315212021494

PC_2 这个48长度的int数组, 其中一个元素由47变为了46

[![](http://qiniu.yujing.icu/typora_img/image-20250315212223689.png)](http://qiniu.yujing.icu/typora_img/image-20250315212223689.png)

image-20250315212223689

参考大佬: [kuaidui作业unidbg逆向（2） - 简书](https://www.jianshu.com/p/c5de1b960931)

> nativeSetToken和nativeInitBaseUtil相比是负责解密数据。

[![](http://qiniu.yujing.icu/typora_img/image-20250316013835840.png)](http://qiniu.yujing.icu/typora_img/image-20250316013835840.png)

image-20250316013835840

对比博文的基本一致, 应该没再修改

[![](http://qiniu.yujing.icu/typora_img/image-20250316154053194.png)](http://qiniu.yujing.icu/typora_img/image-20250316154053194.png)

image-20250316154053194

## hook查看子密钥值

然后对比了子密钥是否一致

[![](http://qiniu.yujing.icu/typora_img/image-20250316202355680.png)](http://qiniu.yujing.icu/typora_img/image-20250316202355680.png)

image-20250316202355680

查看变量`v8`的值, 也就是`x21`

[![](http://qiniu.yujing.icu/typora_img/image-20250316202622626.png)](http://qiniu.yujing.icu/typora_img/image-20250316202622626.png)

image-20250316202622626

但是v8指向的地址, 在循环中被修改了(`v8 += 48`), 一共循环了16次, 所以与原来的地址一共加了 48*16 == 768

[![](http://qiniu.yujing.icu/typora_img/image-20250316203245611.png)](http://qiniu.yujing.icu/typora_img/image-20250316203245611.png)

image-20250316203245611

得到起始地址和长度, 即可查看指定地址的值

[![](http://qiniu.yujing.icu/typora_img/image-20250316203345473.png)](http://qiniu.yujing.icu/typora_img/image-20250316203345473.png)

image-20250316203345473

## 魔改DES总结

根据[twhiteman/pyDes: A pure python module which implements the DES and Triple-DES encryption algorithms.](https://github.com/twhiteman/pyDes)修改

修改点1: PC_2 这个48长度的int数组, 第三十五个元素由47变为了46

修改点2: 密钥位序 由 标准的高位优先（MSB First, 修改为了 **低位优先（LSB First）**

so层签名加解密完成

[![](http://qiniu.yujing.icu/typora_img/image-20250317152213170.png)](http://qiniu.yujing.icu/typora_img/image-20250317152213170.png)

image-20250317152213170