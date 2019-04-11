---
title: 在 Android Library 中使用 R ID 常量
date : 2017.09.06
tag  : Android
---

几个月前实现了一个叫做 SugarAdapter 的库，这是一个优秀的 RecyclerView 脚手架实现，用于支撑公司项目的快速开发，目前还未开源；但是其中使用到了类似 [ButterKnife](https://github.com/JakeWharton/butterknife) 的注解实现来简化代码。随着公司项目越来越大，组件化势在必行，业务代码纷纷拆分到各自的 module 中，此时就遇到了 R.java 中资源 ID 不为常量的问题，具体可以参考 [Non-constant Fields in Case Labels](http://tools.android.com/tips/non-constant-fields).

对于普通的 R ID 调用，例如 switch(menuItem.getItemId()) 可以转换为 if-else; 但是在注解中，例如 ButterKnife 的 @BindView(R.id.view), 由于 R.id.view 不是常量了，会导致编译失败。ButterKnife 的解决方案是使用 Gradle Plugin 将 R.java 拷贝生成一份 R2.java, R2 将 R 中的变量统一加上了 final 变成了常量，这样我们在 module 中使用 @BindView(R2.id.view) 即可。这种解决方案是有问题的，不断有人提交 [issue](https://github.com/JakeWharton/butterknife/issues/762) 反映 findViewById 失败导致 NPE.

究其原因，是因为 R2 仅仅是 module 中 R 的复制，而不是主工程的 R 复制，无法保证 module 的 R2 ID 与主工程的 R ID 一一对应，从而导致 findViewById 失败。可能这个解决方案在 ButterKnife 中是没有问题的，存在特别的处理过程；但是对于我这样自己实现注解绑定的库，想要直接使用 ButterKnife 的 Gradle Plugin 生成的 R2, 还需要绕一些弯子才行。

第一，以 @BindView(R2.id.view) 为例，最终生成的代码不是 findViewById(R2.id.view) 而是 findViewById(0x7f...) 这样的代码。第二，在 module 中 R2 一定是与 R 一一对应的，例如 R2.id.view 一定等于 R.id.view; 但在主工程中却不一定。第三，我们真正需要的是 R 而不是 R2, 使用 R2 只是因为 R ID 在 module 不是常量而已。

综上三点，我们可以想到，通过 0x7f... 找到 R2.id.view 这样的常量名，于是我们也知道了 R.id.view 这样的变量名，于是可以将生成代码的结果从 findViewById(0x7f...) 变成 findViewById(R.id.view), 这里的 R 在主工程的编译过程中会被 inline 成最终确定的数值，从而避免在生成代码的过程中直接填写数值带来的麻烦。在这里推荐一个叫做 [r-parser](https://github.com/HendraAnggrian/r-parser) 的库，可以很方便地实现我们想要的解析过程。