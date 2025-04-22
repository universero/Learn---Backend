___

有种情况我们经常会遇到：某个工作中的项目需要包含并使用另一个项目。 也许是第三方库，或者你独立开发的，用于多个父项目的库。 现在问题来了：你想要把它们当做两个独立的项目，同时又想在一个项目中使用另一个。

Git 通过子模块来解决这个问题。 子模块允许你将一个 Git 仓库作为另一个 Git 仓库的子目录。 它能让你将另一个仓库克隆到自己的项目中，同时还保持提交的独立。

## 创建submodule

`git submodule add url` 即可以当前仓库为主仓库, 将新拉取的仓库作为submodule

父子项目的关系存储在父项目的`.gitmodules` 文件，如果不是新加 submodule，**这个文件通常不必改变了，因为信息比较固定。**

主项目还保存了对应 submodule 的版本号（commit id），**没有冗余存储 submodule 的代码**。

## 更新submodule

 `git submodule update --remote [submodule文件夹相对路径]`

## 操作仓库

可以将主仓库和submodule分开操作, 两者并不影响

## 克隆包含submodule的仓库

### 方法一，按需clone submodule

1. 先`git clone 主项目仓库`并进入主项目文件夹，这时候submodule的文件夹都是空的。
2. 执行`git submodule init [submodule的文件夹的相对路径]`。
3. 执行`git submodule update [submodule的文件夹的相对路径]`。

#### 方法二，一次性clone所有 submodule

1. 先`git clone 主项目仓库`，这时候submodule的文件夹都是空的。
2. 执行`git submodule init`。
3. 执行`git submodule update`。