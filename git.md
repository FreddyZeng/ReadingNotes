仅在分配的 Worktree 中读写项目文件。



不允许主动修改其他 Worktree。



不允许未经授权执行 git reset --hard、git clean -fd、git branch -D 等破坏性操作。



不允许直接修改 main 或自动合并自己的分支。



完成后输出 Commit、Diff、测试结果和未验证事项。



由独立集成流程审查后，才允许合并。
