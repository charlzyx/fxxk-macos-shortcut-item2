# fxxk macos shortcut for iTerm2

因为 iTerm2 菜单占用的快捷键太多了, 跟我抄 NvChad Vim 配置的时候, 有很多冲突, 加上辣鸡麦克的快捷键设置是在反人类

只要写个 node 脚本来批量清除 iTerm2 的快捷键

# 用法
```sh
node index.js
```

# 代码
```js
//  配置自己的保留菜单名称, 不需要父级菜单名称, 只需要自己的
const keep = ['New Tab', "Close"];
// --------------
const fs = require('fs');
const { spawnSync } = require("child_process")

const parse = require('xml-parser');

/* https://github.com/gnachman/iTerm2/blob/master/Interfaces/MainMenu.xib */
const XMLdata = fs.readFileSync('./iTerm2.xib.xml', 'utf-8').toString()
const json = parse(XMLdata);


/** 解析菜单项, 获取快捷键, 不带修饰键 */
const getKey = (menu) => {
  const { attributes = {}, children = [] } = menu;
  if (attributes.keyEquivalent) return attributes.keyEquivalent;
  const maybe = children.find(x => x.attributes && x.attributes.key === 'keyEquivalent');
  if (maybe) {
    return maybe.content;
  }
  return null;
}

/** 解析菜单项, 获取修饰键 */
const getModifier = (menu) => {
  const modifier = {
    shift: false,
    alt: false,
    ctrl: false,
    meta: false
  }
  const key = getKey(menu);

  modifier.shift = /\W/.test(key);

  const modifierMask = menu && menu.children && menu.children.find(x => x.name === 'modifierMask');

  if (modifierMask) {
    const attrs = modifierMask.attributes;
    modifier.alt = attrs.option;
    modifier.ctrl = attrs.control;
    modifier.meta = attrs.command;
    modifier.shift = attrs.shift || modifier.shift;
  }

  const keys = [
    modifier.meta ? '⌘ Command' : null,
    modifier.ctrl ? '⌃ Control' : null,
    modifier.alt ? '⌥ Option/Alt' : null,
    modifier.shift ? '⇧ Shift' : null,
    key
  ].filter(Boolean).join(' + ')

  return keys
}

/** 解析 iTerm.xib.xml 抽取菜单选项 */
const parserXibJson = (menuJson) => {
  const menus = [];
  JSON.stringify(menuJson, (k, v) => {
    const attrs = v.attributes;
    const maybeKey = getKey(v)
    if (maybeKey && attrs.identifier) {
      menus.push({ name: attrs.title, key: maybeKey, modifier: getModifier(v) });
    }
    return v;
  }, 2);
  return menus;
}


/** 清理所有快捷键, 可以指定 title 列表保留 */
const clear = (menuList, keep = []) => {

  const keeper = keep.reduce((map, name) => {
    map[name] = true;
    return map;
  }, {})

  const menus = menuList.filter(x => !keeper[x]);

  console.log('clear shortcut of ', menus);

  // 转换为 defaults write dict 格式
  const dict = menus.map(m => {
    return `"${m}" "\U0000" `;
  }).join(" ")

  // 读取原有值
  const read = `defaults read com.googlecode.iterm2.plist NSUserKeyEquivalents`
  // 清空所有用户配置
  const empty = `defaults delete com.googlecode.iterm2.plist NSUserKeyEquivalents`
  // 写入要覆盖的菜单
  const cli = `defaults write com.googlecode.iterm2.plist NSUserKeyEquivalents -dict ${dict}`

  console.log('------prev dict--------')
  console.log("RUN CLI:: ", read)
  spawnSync(read, { shell: true, stdio: 'inherit' });
  console.log('------empty dict--------')
  console.log("RUN CLI:: ", empty)
  spawnSync(empty, { shell: true, stdio: 'inherit' });
  console.log('------write dict--------')
  console.log("RUN CLI:: ", cli)
  spawnSync(cli, { shell: true, stdio: 'inherit' });
  console.log('------read new dict--------')
  spawnSync(read, { shell: true, stdio: 'inherit' });
}

const menus = parserXibJson(json);

// do clear
clear(menus.map(x => x.name), keep)

```

# Reference

- [Nvchad](https://nvchad.github.io/)
- [macOS defaults](https://macos-defaults.com/)
- [iTerm2/Interfaces/MainMenu.xib](https://github.com/gnachman/iTerm2/blob/master/Interfaces/MainMenu.xib)
