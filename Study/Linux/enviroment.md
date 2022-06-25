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
# Time statistics through time command 

# /usr/bin/time  统计时间

# ===================x86_64-gcc===================
make O=build/x86_64-gcc ARCH=x86_64 defconfig

make O=build/x86_64-gcc ARCH=x86_64 -k -j 16  # bzImage modules
# 1625.66user 161.49system 2:05.83elapsed 1420%CPU (0avgtext+0avgdata 267164maxresident)k
# 0inputs+1338360outputs (3247major+52931761minor)pagefaults 0swaps

make O=build/x86_64-gcc ARCH=x86_64 -k -j 1 # bzImage modules
# 803.77user 72.43system 14:36.00elapsed 100%CPU (0avgtext+0avgdata 268636maxresident)k
# 864inputs+1343600outputs (1141major+53758107minor)pagefaults 0swaps

scripts/clang-tools/gen_compile_commands.py -d build/x86_64-gcc -o build/x86_64-gcc/compile_commands.json

# ===================x86_64-llvm===================
make O=build/x86_64-llvm ARCH=x86_64 LLVM=1 defconfig

make O=build/x86_64-llvm ARCH=x86_64 LLVM=1 -k -j 16  # bzImage modules
# 2240.23user 196.08system 2:48.14elapsed 1448%CPU (0avgtext+0avgdata 260396maxresident)k
# 0inputs+1355624outputs (181121major+58080691minor)pagefaults 0swaps

make O=build/x86_64-llvm ARCH=x86_64 LLVM=1 -k -j 1  # bzImage modules
# 1185.61user 85.61system 21:10.51elapsed 100%CPU (0avgtext+0avgdata 263212maxresident)k
# 280inputs+1363240outputs (80833major+49733819minor)pagefaults 0swaps
scripts/clang-tools/gen_compile_commands.py -d build/x86_64-llvm -o build/x86_64-llvm/compile_commands.json

# ===================riscv-gcc===================
make O=build/riscv-gcc ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- defconfig
make O=build/riscv-gcc ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -k -j 16 # Image.gz modules
# 956.17user 83.02system 1:14.13elapsed 1401%CPU (0avgtext+0avgdata 201624maxresident)k
# 0inputs+660408outputs (1396major+27695853minor)pagefaults 0swaps
scripts/clang-tools/gen_compile_commands.py -d build/riscv-gcc -o build/riscv-gcc/compile_commands.json

# ===================riscv-llvm===================   报错
# CROSS_COMPILE=riscv64-linux-gnu-
make O=build/riscv-llvm ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- CC=clang defconfig
make O=build/riscv-llvm ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- CC=clang -k -j 16 # Image.gz modules
scripts/clang-tools/gen_compile_commands.py -d build/arm64 -o build/arm64

ln -sf build/riscv-gcc/compile_commands.json .

# ===================arm64-llvm===================
make O=build/arm64-llvm ARCH=arm64 LLVM=1 defconfig
make O=build/arm64-llvm ARCH=arm64 LLVM=1 -k -j 16 # Image.gz modules
scripts/clang-tools/gen_compile_commands.py -d build/arm64-llvm -o build/arm64-

ln -sf build/riscv-gcc/compile_commands.json .
```

