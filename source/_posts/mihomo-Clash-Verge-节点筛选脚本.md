---
title: mihomo(Clash Verge) 节点筛选脚本
date: 2026/5/15 16:10:00
---

## 前言
- 节点数量过多, 导致软件开启会卡几秒, 因此做了个筛选脚本
## 使用方法
- 打开 `Clash Verge`
- 侧边栏选择 `订阅`
- 找到需要筛选的节点右键选择 `扩展脚本`
- 把脚本复制进去, 按需修改变量, 保存
## 脚本
``` js
/**
 * Clash 配置文件预处理脚本
 *
 * @param {Object} config - 原始配置对象
 * @param {string} profileName - 配置文件名称
 * @returns {Object} 处理后的配置对象
 */
function main(config, profileName) {
  const BUILTIN_IN_GROUP = new Set(["DIRECT", "REJECT", "REJECT-DROP", "PASS", "COMPATIBLE"]);

  // 任一项设为 null / undefined / '' 即关闭；关闭 include = 全保留，关闭 exclude = 不排除
  const includeFilters = /SG|JP|HK|3x US/i;
  const excludeFilters = /5x|10x/i;

  function toRegExp(pattern) {
    if (pattern == null || pattern === "") return null;
    if (pattern instanceof RegExp) return pattern;
    try {
      return new RegExp(String(pattern));
    } catch {
      return null;
    }
  }

  /** @param {RegExp | string | null | undefined} pattern */
  function passesInclude(name, pattern) {
    const re = toRegExp(pattern);
    if (!re || re.source === "") return true;
    try {
      return re.test(name);
    } catch {
      return true;
    }
  }

  /** @param {RegExp | string | null | undefined} pattern */
  function hitsExclude(name, pattern) {
    const re = toRegExp(pattern);
    if (!re || re.source === "") return false;
    try {
      return re.test(name);
    } catch {
      return false;
    }
  }

  const filteredProxies = config.proxies.filter((proxy) => {
    return passesInclude(proxy.name, includeFilters) && !hitsExclude(proxy.name, excludeFilters);
  });

  const filteredProxyNames = new Set(filteredProxies.map((p) => p.name));

  config.proxies = filteredProxies;

  if (config["proxy-groups"]) {
    const groupNames = new Set(config["proxy-groups"].map((g) => g.name));

    config["proxy-groups"].forEach((group) => {
      if (group.proxies) {
        group.proxies = group.proxies.filter((pName) => {
          if (BUILTIN_IN_GROUP.has(pName)) return true;
          const isAnotherGroup = groupNames.has(pName);
          const isExistingProxy = filteredProxyNames.has(pName);
          return isAnotherGroup || isExistingProxy;
        });

        if (group.proxies.length === 0) {
          group.proxies.push("DIRECT");
        }
      }
    });
  }

  return config;
}
```
