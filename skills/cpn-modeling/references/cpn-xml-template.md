# CPN XML 模板参考（CPN Tools 4.x 兼容）

## 完整模板

```xml
<?xml version="1.0" encoding="utf-8"?>
<workspaceElements>
  <cpnet>
    <globbox>
      <block id="ID1">
        <id>ColorSets</id>
        <color id="ID2">
          <id>SubProjectType</id>
          <enum>
            <id>新建</id>
            <id>水电</id>
            <id>瓦工</id>
            <id>木工</id>
            <id>油漆</id>
            <id>安装</id>
          </enum>
        </color>
        <color id="ID3">
          <id>ChainType</id>
          <enum>
            <id>SAL</id>
            <id>PKG</id>
            <id>MPK</id>
            <id>CPK</id>
          </enum>
        </color>
      </block>
    </globbox>

    <page id="PAGE1">
      <pageattr name="新建阶段"/>
      <place id="P1">
        <posattr x="100" y="100"/>
        <text>TaskDA_待完成</text>
        <type><id>SubProjectType</id></type>
        <initmark><text>empty</text></initmark>
      </place>
      <place id="P2">
        <posattr x="300" y="100"/>
        <text>TaskDA_完成</text>
        <type><id>SubProjectType</id></type>
        <initmark><text>empty</text></initmark>
      </place>
      <trans id="T1">
        <posattr x="200" y="100"/>
        <text>完成TaskDA</text>
        <condition><text>true</text></condition>
      </trans>
      <arc id="A1" orientation="PtoT">
        <transend idref="T1"/>
        <placeend idref="P1"/>
        <annot><text>1`新建</text></annot>
      </arc>
      <arc id="A2" orientation="TtoP">
        <transend idref="T1"/>
        <placeend idref="P2"/>
        <annot><text>1`新建</text></annot>
      </arc>
    </page>

    <page id="PAGE2">
      <pageattr name="水电阶段"/>
      <place id="P3">
        <posattr x="100" y="100"/>
        <text>新建_TaskDA_完成_融合</text>
        <type><id>SubProjectType</id></type>
        <initmark><text>empty</text></initmark>
      </place>
      <trans id="T2">
        <posattr x="200" y="100"/>
        <text>发起MaterialsSI</text>
        <condition><text>true</text></condition>
      </trans>
      <arc id="A3" orientation="PtoT">
        <transend idref="T2"/>
        <placeend idref="P3"/>
        <annot><text>1`新建</text></annot>
      </arc>
    </page>

  </cpnet>
</workspaceElements>
```

## 关键规则

- 每个子项目对应一个 `<page>`
- colorset 统一定义在 `<globbox>` 的 `<block>` 内
- 跨子项目 FS 规则通过**融合库所（fusion place）**表达：两个 page 中同名库所共享 token
- 跨交易链 FS 规则通过变迁的 `<condition>` 守卫条件表达
- arc 的 `orientation` 属性：`PtoT` = 输入弧，`TtoP` = 输出弧
