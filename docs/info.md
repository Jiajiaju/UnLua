# 将 IDEA 作为 Lua IDE

## 代码提示
1. 下载安装 IDEA；
2. 安装 EmmyLua 插件；
3. **Plugins/UnLua/Source/UnLuaIntelliSense.Build.cs** 文件中第 51 行修改为
```c
PublicDefinitions.Add("ENABLE_INTELLISENSE=1");
```
4. 删除整个 **Plugins/UnLua/Intermediate** 目录后重新构建；
5. 构建完成后，**Plugins/UnLua/Intermediate/IntelliSense** 目录下会生成 Lua 代码提示文件；
6. 将 **Plugins/UnLua/Intermediate/IntelliSense** 打包 **zip**；
7. IEDA 将 **Content/Script** 作为一个工程打开，然后打开工程配置（Porject Structure）选择 Modules——Dependencise，点击“Add”加号，选择 Library——Lua Zip Library，然后选择 6 中打包的 zip 文件，将其以依赖包的形式导入工程成中。
8. apply 后，等待索引完毕即可使用。

## 调试