# Enviroment

### preparation

```bash
# use make help to get detail info for building the kernelen
make help
vi /Documentation/admin-guide/README.rst

yay -S lld
```



```bash
cd ${linux kernel project}
make O=build/x86 ARCH=x86_64 LLVM=1 defconfig
make O=build/x86 ARCH=x86_64 LLVM=1 -k -j 16 bzImage modules
scripts/clang-tools/gen_compile_commands.py -d build/x86 -o build/x86

make O=build/arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LLVM=1 defconfig
make O=build/arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LLVM=1 -k Image.gz modules
scripts/clang-tools/gen_compile_commands.py -d build/arm64 -o build/arm64
```

