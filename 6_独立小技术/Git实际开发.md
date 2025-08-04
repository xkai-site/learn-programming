# Git实际开发

1. 在公司中的仓库克隆代码
   ```bash
   git clone [仓库地址]
   cd [项目本地地址]
   ```

2. 切换分支，一般来说公司会有master和dev等分支，开发人员使用dev
   ```bash
   git checkout dev          # 切换到dev分支
   git pull origin dev       # 获取最新代码
   ```

3. 建议创建一个分支给自己进行开发
   ```bash
   git checkout -b feature/your-feature-name origin/dev
   ```

4. 日常开发
   ```bash
   # 1. 开发前先同步最新代码
   git fetch origin                  # 先获取所有远程变更
   git merge origin/dev              # 直接合并远程dev到当前分支（避免本地dev未更新问题）
   
   
   # 2. 在你的分支开发提交
   git checkout feature/your-branch   # 切换回你的功能分支
   git merge dev                      # 将本地dev的更新合并到当前功能分支
   
   #3. 正常开发
   git add .
   git commit -m "描述你的修改"
   
   # 4. 定期推送到远程（第一次推送需要设置上游）
   git push -u origin feature/your-feature-name
   
   # 5. 开发完成后创建Pull Request/Merge Request到dev分支
   ```