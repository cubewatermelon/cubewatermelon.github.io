<html>
<body>
<!--StartFragment--><!DOCTYPE html><h1 cid="n0" mdtype="heading" class="md-end-block md-heading md-focus" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 2.25em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.2; cursor: text; border-bottom: 1px solid rgb(238, 238, 238); color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain md-expand" style="box-sizing: border-box;">DeepSeek Harness 插件化机制、缓存与推理回传解析</span></h1><blockquote cid="n2" mdtype="blockquote" style="box-sizing: border-box; margin: 0.8em 0px; border-left: 4px solid rgb(223, 226, 229); padding: 0px 15px; color: rgb(119, 119, 119); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><p cid="n3" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px; white-space: pre-wrap; position: relative;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">本文整理自一次关于 DeepSeek Harness（DSH）框架的深入问答，内容均基于框架源码（</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">node_modules/@deepseek-ai/*</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">）核实的事实，重点覆盖三大主题：</span><span md-inline="softbreak" class="md-softbreak" style="box-sizing: border-box;">
</span><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">① Agent 插件化架构 ② KV 缓存命中机制 ③ 思考模式下的推理回传（reasoning passback）</span></strong></span></p></blockquote><div tabindex="-1" cid="n4" mdtype="hr" class="md-hr md-end-block" style="box-sizing: border-box; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><hr style="box-sizing: content-box; height: 2px; margin: 16px 0px; border: 0px none; padding: 0px; background-color: rgb(231, 231, 231); overflow: hidden;"></div><h2 cid="n5" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.75em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.225; cursor: text; border-bottom: 1px solid rgb(238, 238, 238); color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">目录</span></h2><ol class="ol-list" start="" cid="n6" mdtype="list" style="box-sizing: border-box; margin: 0.8em 0px; padding-left: 30px; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><li class="md-list-item" cid="n7" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n8" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E4%B8%80%E4%BC%9A%E8%AF%9D%E7%8E%AF%E5%A2%83%E4%B8%8E%E8%83%8C%E6%99%AF" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">会话环境与背景</span></a></span></p></li><li class="md-list-item" cid="n9" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n10" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E4%BA%8Cagent-%E9%A2%84%E8%AE%BEptc-%E6%A8%A1%E5%BC%8F%E4%B8%8E%E5%85%B6%E4%BB%96%E5%86%85%E7%BD%AE%E6%A8%A1%E5%BC%8F" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">Agent 预设：PTC 模式与其他内置模式</span></a></span></p></li><li class="md-list-item" cid="n11" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n12" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E4%B8%89%E6%8F%92%E4%BB%B6%E5%8C%96%E6%9E%B6%E6%9E%84%E6%9C%80%E5%B0%8F%E6%A0%B8%E5%BF%83--%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%8F%92%E4%BB%B6" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">插件化架构：最小核心 + 事件驱动插件</span></a></span></p></li><li class="md-list-item" cid="n13" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n14" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E5%9B%9B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%8E%8B%E7%BC%A9%E4%B8%80%E4%B8%AA%E5%85%B8%E5%9E%8B%E7%9A%84%E6%8F%92%E4%BB%B6%E4%BD%93%E7%B3%BB" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">上下文压缩：一个典型的插件体系</span></a></span></p></li><li class="md-list-item" cid="n15" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n16" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E4%BA%94%E8%BD%A8%E8%BF%B9trajectory%E4%BC%9A%E8%AF%9D%E8%BF%87%E7%A8%8B%E7%9A%84%E5%AE%8C%E6%95%B4%E6%A1%A3%E6%A1%88" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">轨迹（Trajectory）：会话过程的完整档案</span></a></span></p></li><li class="md-list-item" cid="n17" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n18" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E5%85%ADkv-%E7%BC%93%E5%AD%98%E5%91%BD%E4%B8%AD%E6%9C%BA%E5%88%B6" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">KV 缓存命中机制</span></a></span></p></li><li class="md-list-item" cid="n19" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n20" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E4%B8%83%E6%8E%A8%E7%90%86%E5%9B%9E%E4%BC%A0reasoning-passback" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">推理回传（Reasoning Passback）</span></a></span></p></li><li class="md-list-item" cid="n21" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n22" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="link" class="md-meta-i-c  md-link" style="box-sizing: border-box;"><a class="md-inner-link" href="#%E5%85%AB%E6%80%BB%E7%BB%93" style="box-sizing: border-box; cursor: pointer; color: rgb(65, 131, 196); -webkit-user-drag: none;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">总结</span></a></span></p></li></ol><div tabindex="-1" cid="n23" mdtype="hr" class="md-hr md-end-block" style="box-sizing: border-box; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><hr style="box-sizing: content-box; height: 2px; margin: 16px 0px; border: 0px none; padding: 0px; background-color: rgb(231, 231, 231); overflow: hidden;"></div><h2 cid="n24" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.75em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.225; cursor: text; border-bottom: 1px solid rgb(238, 238, 238); color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">一、会话环境与背景</span></h2><ul class="ul-list" cid="n25" mdtype="list" data-mark="-" style="box-sizing: border-box; margin: 0.8em 0px; padding-left: 30px; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><li class="md-list-item" cid="n26" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n27" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">运行框架</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：DeepSeek Harness（DSH）智能体框架</span></p></li><li class="md-list-item" cid="n28" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n29" mdtype="paragraph" class="md-end-block md-p md-focus" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">当前模型</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">deepseek-v4-flash</code></span></p></li><li class="md-list-item" cid="n32" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n33" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">能力清单</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：文件读写、PowerShell、检索（文件/网页）、Skills、todo/goal 计划、子代理、后台任务、工作流等</span></p></li></ul><p cid="n34" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0.8em 0px; white-space: pre-wrap; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">本次对话从环境认知开始，经历了 LightRAG 与知识图谱文档对比、PTC 模式探源、轨迹（trajectory）解析、插件架构（loop、compaction）、缓存命中机制，最后深入推理回传机制。本文聚焦后四个技术主题。</span></p><div tabindex="-1" cid="n35" mdtype="hr" class="md-hr md-end-block" style="box-sizing: border-box; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><hr style="box-sizing: content-box; height: 2px; margin: 16px 0px; border: 0px none; padding: 0px; background-color: rgb(231, 231, 231); overflow: hidden;"></div><h2 cid="n36" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.75em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.225; cursor: text; border-bottom: 1px solid rgb(238, 238, 238); color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">二、Agent 预设：PTC 模式与其他内置模式</span></h2><h3 cid="n37" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.5em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.43; cursor: text; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">2.1 预设即"插件组装"</span></h3><p cid="n38" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0.8em 0px; white-space: pre-wrap; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">DSH 官方定义：</span></p><blockquote cid="n39" mdtype="blockquote" style="box-sizing: border-box; margin: 0.8em 0px; border-left: 4px solid rgb(223, 226, 229); padding: 0px 15px; color: rgb(119, 119, 119); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><p cid="n40" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">预设（Agent preset）即一个会话的 Agent 所运行的插件组装——它的工具、提示词与能力。</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;"> 复制一份既有预设改成自己的，或用「创造模式」让 Agent 帮你创建。</span></p></blockquote><p cid="n41" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0.8em 0px; white-space: pre-wrap; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">预设对"此后新建的会话生效，运行中的会话保持它开始时的预设"。</span></p><h3 cid="n42" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.5em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.43; cursor: text; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">2.2 四种内置预设</span></h3><figure class="md-table-fig table-figure" cid="n43" mdtype="table" style="box-sizing: border-box; margin: 1.2em 0px; overflow-x: auto; max-width: calc(100% + 16px); padding: 0px; cursor: default; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;">
预设 | 说明
-- | --
标准模式 | 功能完整的编码 Agent：文件编辑、Shell、文件/网页检索、Skills、计划、目标、子代理、工作流
PTC 模式（Code mode） | 标准模式全部能力 + Code Mode SDK：工具以异步绑定形式暴露，模型可写一段 TypeScript 程序（支持顶层 await/return）在一次运行中组合多步操作
极简模式 | 仅双工具：持久 bash + str_replace_editor，无上下文压缩
创造模式 | 标准模式能力 + 运行时检查、插件实验、预设创作指导

</figure><p cid="n361" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0.8em 0px; white-space: pre-wrap; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">一句话概括</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：</span></p><blockquote cid="n362" mdtype="blockquote" style="box-sizing: border-box; margin: 0.8em 0px; border-left: 4px solid rgb(223, 226, 229); padding: 0px 15px; color: rgb(119, 119, 119); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><p cid="n363" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px; white-space: pre-wrap; position: relative;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">普通回复是"思考的终点"——推理使命已完成，产出已交付，下一轮从头思考即可，回传纯属浪费；工具调用是"思考的中断"——模型话没说完，推理必须保留下来，才能在工具结果回来后接着想，这是协议要求，也是思考连续性的需要。</span></p></blockquote><h3 cid="n364" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.5em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.43; cursor: text; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">7.5 与缓存的关系</span></h3><ul class="ul-list" cid="n365" mdtype="list" data-mark="-" style="box-sizing: border-box; margin: 0.8em 0px; padding-left: 30px; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><li class="md-list-item" cid="n366" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n367" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">推理回传在工具往返期间</span><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">追加</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：它附加在请求前缀尾部（该条 assistant 消息内部），</span><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">不破坏</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">更早部分（系统提示词、历史）的缓存复用；</span></p></li><li class="md-list-item" cid="n368" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n369" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">推理内容本身每次是新的（每轮思考不同），属于 cache miss，但它后面紧跟的 tool 结果等又是追加的；</span></p></li><li class="md-list-item" cid="n370" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n371" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">无工具调用的轮次丢弃推理 → 历史更精简、前缀更稳定、不产生多余 token 成本。</span></p></li></ul><div tabindex="-1" cid="n372" mdtype="hr" class="md-hr md-end-block" style="box-sizing: border-box; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><hr style="box-sizing: content-box; height: 2px; margin: 16px 0px; border: 0px none; padding: 0px; background-color: rgb(231, 231, 231); overflow: hidden;"></div><h2 cid="n373" mdtype="heading" class="md-end-block md-heading" style="box-sizing: border-box; white-space: pre-wrap; break-after: avoid-page; break-inside: avoid; orphans: 4; font-size: 1.75em; margin-top: 1rem; margin-bottom: 1rem; position: relative; font-weight: bold; line-height: 1.225; cursor: text; border-bottom: 1px solid rgb(238, 238, 238); color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">八、总结</span></h2><ol class="ol-list" start="" cid="n374" mdtype="list" style="box-sizing: border-box; margin: 0.8em 0px; padding-left: 30px; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><li class="md-list-item" cid="n375" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n376" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s" style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">插件化</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：DSH 采用"最小核心 loop + 事件驱动插件"架构。loop 是唯一含具体循环逻辑的插件，其余一切策略（压缩、重试、权限、子代理、持久化、UI）都是挂在其事件体系上的外围插件；Agent 预设（标准/PTC/极简/创造）本质是插件组装方案。</span></p></li><li class="md-list-item" cid="n377" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n378" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">能力 seam 模式</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：以压缩为例——接口（</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-compaction</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">）与实现（</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-compaction-basic</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">）分离，接口只规定"做什么"，实现决定"怎么做"，两者可独立替换；替换入口是 preset 的 </span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">agent.cordis.yml</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">。</span></p></li><li class="md-list-item" cid="n379" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n380" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">缓存</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：高命中率来自框架把请求前缀做"字节级稳定"（只追加历史、工具目录跨模式稳定、压缩回放前缀、排除噪声），命中主体是系统提示词 + 工具 schema + 历史消息前缀；命中由 DeepSeek 服务端前缀缓存自动完成，框架不主动"做"缓存，只是"省"出来的。</span></p></li><li class="md-list-item" cid="n381" mdtype="list_item" style="box-sizing: border-box; margin: 0px; position: relative;"><p cid="n382" mdtype="paragraph" class="md-end-block md-p" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0px 0px 0.5rem; white-space: pre-wrap; position: relative;"><span md-inline="strong" class="md-pair-s " style="box-sizing: border-box;"><strong style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">推理回传</span></strong></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">：思考模式下工具调用轮必须回传 </span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">reasoning_content</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">（协议强制 + 思考连续性），普通回复轮丢弃推理（终态无需续接 + 省 token）；DSH 适配器精确执行这条规则，并以"丢弃"策略优化成本与缓存。</span></p></li></ol><div tabindex="-1" cid="n383" mdtype="hr" class="md-hr md-end-block" style="box-sizing: border-box; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; orphans: 2; text-align: start; text-indent: 0px; text-transform: none; white-space: normal; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><hr style="box-sizing: content-box; height: 2px; margin: 16px 0px; border: 0px none; padding: 0px; background-color: rgb(231, 231, 231); overflow: hidden;"></div><p cid="n384" mdtype="paragraph" class="md-end-block md-p md-focus" style="box-sizing: border-box; line-height: inherit; orphans: 4; margin: 0.8em 0px; white-space: pre-wrap; position: relative; color: rgb(51, 51, 51); font-family: &quot;Open Sans&quot;, &quot;Clear Sans&quot;, &quot;Helvetica Neue&quot;, Helvetica, Arial, &quot;Segoe UI Emoji&quot;, sans-serif; font-size: 18px; font-style: normal; font-variant-ligatures: normal; font-variant-caps: normal; font-weight: 400; letter-spacing: normal; text-align: start; text-indent: 0px; text-transform: none; widows: 2; word-spacing: 0px; -webkit-text-stroke-width: 0px; text-decoration-thickness: initial; text-decoration-style: initial; text-decoration-color: initial;"><span md-inline="em" class="md-pair-s md-expand" style="box-sizing: border-box;"><em style="box-sizing: border-box;"><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">文档基于 DeepSeek Harness 框架源码核实整理，主要来源：</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-agent-loop</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">、</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-compaction(-basic/-tool-result-pruner)</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">、</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-llm-deepseek</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">、</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh-client-ui-trajectory</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;">、</span><span md-inline="code" spellcheck="false" class="md-pair-s" style="box-sizing: border-box;"><code style="box-sizing: border-box; font-family: var(--monospace); text-align: left; vertical-align: initial; border: 1px solid rgb(231, 234, 237); background-color: rgb(243, 244, 244); border-radius: 3px; padding: 0px 2px; font-size: 0.9em;">dsh/config/agent-presets/*</code></span><span md-inline="plain" class="md-plain" style="box-sizing: border-box;"> 等包的 README 与源码。</span></em></span></p><!--EndFragment-->
</body>
</html># DeepSeek Harness 插件化机制、缓存与推理回传解析

> 本文整理自一次关于 DeepSeek Harness（DSH）框架的深入问答，内容均基于框架源码（`node_modules/@deepseek-ai/*`）核实的事实，重点覆盖三大主题：
> **① Agent 插件化架构 ② KV 缓存命中机制 ③ 思考模式下的推理回传（reasoning passback）**

---

## 目录

1. [[会话环境与背景](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E4%B8%80%E4%BC%9A%E8%AF%9D%E7%8E%AF%E5%A2%83%E4%B8%8E%E8%83%8C%E6%99%AF)](#一会话环境与背景)
2. [[Agent 预设：PTC 模式与其他内置模式](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E4%BA%8Cagent-%E9%A2%84%E8%AE%BEptc-%E6%A8%A1%E5%BC%8F%E4%B8%8E%E5%85%B6%E4%BB%96%E5%86%85%E7%BD%AE%E6%A8%A1%E5%BC%8F)](#二agent-预设ptc-模式与其他内置模式)
3. [[插件化架构：最小核心 + 事件驱动插件](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E4%B8%89%E6%8F%92%E4%BB%B6%E5%8C%96%E6%9E%B6%E6%9E%84%E6%9C%80%E5%B0%8F%E6%A0%B8%E5%BF%83--%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%E6%8F%92%E4%BB%B6)](#三插件化架构最小核心--事件驱动插件)
4. [[上下文压缩：一个典型的插件体系](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E5%9B%9B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%8E%8B%E7%BC%A9%E4%B8%80%E4%B8%AA%E5%85%B8%E5%9E%8B%E7%9A%84%E6%8F%92%E4%BB%B6%E4%BD%93%E7%B3%BB)](#四上下文压缩一个典型的插件体系)
5. [[轨迹（Trajectory）：会话过程的完整档案](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E4%BA%94%E8%BD%A8%E8%BF%B9trajectory%E4%BC%9A%E8%AF%9D%E8%BF%87%E7%A8%8B%E7%9A%84%E5%AE%8C%E6%95%B4%E6%A1%A3%E6%A1%88)](#五轨迹trajectory会话过程的完整档案)
6. [[KV 缓存命中机制](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E5%85%ADkv-%E7%BC%93%E5%AD%98%E5%91%BD%E4%B8%AD%E6%9C%BA%E5%88%B6)](#六kv-缓存命中机制)
7. [[推理回传（Reasoning Passback）](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E4%B8%83%E6%8E%A8%E7%90%86%E5%9B%9E%E4%BC%A0reasoning-passback)](#七推理回传reasoning-passback)
8. [[总结](https://github.com/cubewatermelon/cubewatermelon.github.io/issues/61#%E5%85%AB%E6%80%BB%E7%BB%93)](#八总结)

---

## 一、会话环境与背景

- **运行框架**：DeepSeek Harness（DSH）智能体框架
- **当前模型**：`deepseek-v4-flash`
- **能力清单**：文件读写、PowerShell、检索（文件/网页）、Skills、todo/goal 计划、子代理、后台任务、工作流等

本次对话从环境认知开始，经历了 LightRAG 与知识图谱文档对比、PTC 模式探源、轨迹（trajectory）解析、插件架构（loop、compaction）、缓存命中机制，最后深入推理回传机制。本文聚焦后四个技术主题。

---

## 二、Agent 预设：PTC 模式与其他内置模式

### 2.1 预设即"插件组装"

DSH 官方定义：

> **预设（Agent preset）即一个会话的 Agent 所运行的插件组装——它的工具、提示词与能力。** 复制一份既有预设改成自己的，或用「创造模式」让 Agent 帮你创建。

预设对"此后新建的会话生效，运行中的会话保持它开始时的预设"。

### 2.2 四种内置预设

| 预设                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| **标准模式**              | 功能完整的编码 Agent：文件编辑、Shell、文件/网页检索、Skills、计划、目标、子代理、工作流 |
| **PTC 模式**（Code mode） | 标准模式全部能力 + **Code Mode SDK**：工具以异步绑定形式暴露，模型可写一段 TypeScript 程序（支持顶层 `await`/`return`）在一次运行中组合多步操作 |
| **极简模式**              | 仅双工具：持久 bash + str_replace_editor，无上下文压缩       |
| **创造模式**              | 标准模式能力 + 运行时检查、插件实验、预设创作指导            |

PTC 模式在中文 UI 中的全称是"PTC 模式"，英文为 "Code mode"；其核心是 `@deepseek-ai/dsh-code-runtime`（目前基于 Node worker 线程后端），对外通过 `run_code` 工具暴露。**每个预设本质上是一套不同的插件组装方案**。

---

## 三、插件化架构：最小核心 + 事件驱动插件

### 3.1 loop 是一个插件——而且是"唯一不可替换的核心插件"

`@deepseek-ai/dsh-agent-loop` 的 package.json 描述：

> **"The concrete agent loop plugin for the DeepSeek Harness"**

其 README 强调：

> **这是 harness 中唯一包含具体循环逻辑的包。** 其他所有内容要么是抽象服务，要么是针对扩展点的插件：新行为应放入插件，而不是这里。

loop 的职责被刻意保持最小——**调用模型、运行工具、重复**（call model, run tools, repeat），并驱动**会话（session）→ 轮次（turn）→ 步骤（step）**的生命周期：

| 层面       | 职责                                                         |
| ---------- | ------------------------------------------------------------ |
| 核心循环   | 调用模型 → 运行工具 → 重复                                   |
| 服务接口   | `AgentLoop`（ctx 键 `agentLoop`），实现 `AgentFactory`（`ctx.agents.create()/resume()`） |
| inbox 机制 | `followup` / `steer` / `inject` 三种输入队列，控制唤醒与下一步 |
| 并发       | 独占调用形成屏障，并行安全调用用有界滚动池（默认 10）        |
| 模型适配   | `ctx.llm.prepareCall()`、适配器默认值标记、重试策略          |

### 3.2 一切策略能力都是外围插件

| 能力               | 实现方式                                                     |
| ------------------ | ------------------------------------------------------------ |
| 上下文压缩         | 插件监听 `agent/pre-step`（压力）、`agent/request-error`（溢出修复） |
| 模型请求重试       | `dsh-llm-retry` 监听 `agent/request-error`，退避重试         |
| 沙箱/权限/计划模式 | `tools/pre-execute` 拒绝或询问、`tools.guard()` 策略、`tools/post-execute` 处理结果 |
| 子代理             | **在循环外部**：`ctx.subagents` 提供方用 `ctx.agents.create()` 创建 agent |
| 持久化             | 监听 `session/event` 延后写入，`session/flush` 显式屏障      |
| UI                 | 监听 `session/event`（token 流、工具活动）+ `agent/*` 控制事件 |

**架构哲学**：loop 只提供事件体系（`agent/*`、`tools/*`、`session/*`），所有策略（目标、计划、子代理、权限、重试、压缩）都是挂在这副骨架上的外围插件。这正是 DSH 能自由组合出标准/PTC/极简/创造等预设的原因。

---

## 四、上下文压缩：一个典型的插件体系

### 4.1 "三包一体"的分层结构（capability seam 模式）

| 包                                               | 角色                             | 职责                                                         |
| ------------------------------------------------ | -------------------------------- | ------------------------------------------------------------ |
| `@deepseek-ai/dsh-compaction`                    | **Service Definition**（接口层） | 定义 `CompactionEngine` 抽象服务（`ctx.compaction`）、`compaction/*` 事件、`CompactionResult`、工具配对边界 helper。**只规定"压缩做什么"** |
| `@deepseek-ai/dsh-compaction-basic`              | **Service Provider**（默认实现） | `ctx.tokenMeter` 压力测量、阈值/保留尾部预算、`llm.stream()` 摘要、溢出恢复。**规定"怎么做"** |
| `@deepseek-ai/dsh-command-compact`               | **Consumer**                     | 面向用户的 `/compact` 手动压缩命令                           |
| `@deepseek-ai/dsh-compaction-tool-result-pruner` | **配套服务**（可选）             | 无模型的工具结果剪枝：把超大的 `tool/result` 改写为"头部 + 省略标记 + 尾部" |

关键设计：

- 接口与实现**可独立演进、独立替换**（Service Definition 只依赖 `dsh-session` 和 `dsh-llm`，不依赖具体后端）
- `tokenMeter`（token 计量）**刻意不在**压缩的 realm 里——它常驻 host 平面，所有会话共享（浏览器 UI 每时每刻在读它的计量结果）

### 4.2 压缩的组装位置（agent.cordis.yml）

压缩插件的组装点在每个 Agent preset 的 `agent.cordis.yml`。以标准模式为例：

```yaml
# ── compaction ──────────────────────────────────────────
- id: compaction
  name: cordis:group
  group: true
  isolate:
    compaction: true
    toolResultPruner: true
  config:
    - id: compaction-basic
      name: '@deepseek-ai/dsh-compaction-basic'
    - id: command-compact
      name: '@deepseek-ai/dsh-command-compact'
    - id: tool-result-pruner
      name: '@deepseek-ai/dsh-compaction-tool-result-pruner'
      config:
        thresholdChars: 8192
        headChars: 4096
        tailChars: 1024
```

### 4.3 如何替换压缩功能（三条路径）

**路径 A：在预设配置里替换（推荐）**

1. Web GUI 的"Agent 预设"设置里复制一份预设；
2. 编辑其 `agent.cordis.yml` 的 `compaction` group：换 `name` 为自定义后端包名、调参或整体删除（极简模式即无压缩）；
3. 新建会话时选择该自定义预设。

`compaction-basic` 关键配置：

| 配置键                                         | 默认   | 含义                                         |
| ---------------------------------------------- | ------ | -------------------------------------------- |
| `thresholdRatio`                               | `0.8`  | 上下文用量达窗口 80% 触发压缩                |
| `retainRatio`                                  | `0.16` | 保留近期表层 16% 逐字内容                    |
| `retainTokens`                                 | —      | 与 `retainRatio` 互斥的绝对保留预算          |
| `summarizationProvider` / `summarizationModel` | 空     | 指定摘要模型（默认复用当前路由）             |
| `maxTokens`                                    | `8192` | 摘要输出上限                                 |
| `auto`                                         | `true` | 是否自动触发（`false` 则仅 `/compact` 手动） |
| `modelPolicies`                                | `[]`   | 按 `{provider, model}` 精确覆盖策略          |

**路径 B：实现自己的压缩后端**
继承 `CompactionEngine`，实现 `compactIfNeeded(agent, trigger, signal)`、`compactNow(agent, signal)`、`compactRegion(start, end, agent, signal?)`，摘要直接走 `ctx.llm.stream()`（不是 loop 步骤），用 `compactCheckpointSource(compactionId)` 创建替换检查点，以插件形式注册为 `ctx.compaction`。

**路径 C：只换摘要策略（最轻量）**
覆盖 `BasicCompactionEngine` 受保护的 `summarize()` 子类钩子——压力测量、保留策略、缩减验证全部由基类负责，只改"怎么总结"（模板摘要器或远程摘要器）。

### 4.4 压缩的表面约定（surface contract）

- 成功压缩 = 用一个 user 角色的摘要检查点替换较早表层范围（`surfaceOp: {op: 'replace', start, end}`），原始事件仍保留在日志中（回放具有确定性）
- 压缩以 `compaction/start`（获取锁）开始、`compaction/end`（释放锁）结束；`compaction/*` 事件仅存在于日志，不出现在表层
- 只有 `user/message`、`assistant/message`、`tool/result` 可以携带 `surfaceOp`
- 模型看到的检查点格式：`<compacted-summary>` 标签包裹的结构化摘要（Primary Request / Key Technical Concepts / Files and Code / Errors and Fixes / Pending Jobs / Current Work / Next Step / Critical Context 等段落）

---

## 五、轨迹（Trajectory）：会话过程的完整档案

### 5.1 记录类型与字段

轨迹记录有 7 种类型（`TrajectoryCellKind`）：`system`、`user`、`context`、`compacted`、`message`、`tool`、`subtool`。

每条记录包含：

| 字段                            | 含义                                                       |
| ------------------------------- | ---------------------------------------------------------- |
| `#N` 序号 + 摘要                | 记录索引与单行摘要（`text` / `previewMarkdown`）           |
| `inputDetail`                   | 输入方向完整内容（对 tool 是调用参数，对 user 是消息原文） |
| `outputDetail`                  | 输出方向完整内容（助手回复 / 工具结果）                    |
| `thinkingDetail`                | 模型的推理内容                                             |
| `sourceBlocks` / `outputBlocks` | 按模型消息顺序保留的原始内容块（文本、图片、tool-call 块） |
| `schemaDetail`                  | 调用那一刻模型看到的工具 schema                            |
| `assistantMetrics`              | 首 token 时间（TTFT）、解码吞吐、token 用量                |
| `startedAt` / `timeSeconds`     | 开始时间与耗时                                             |
| `isError` / `callId`            | 失败标记与调用关联 ID                                      |

### 5.2 tool 部分的 payload 是什么

**payload = `argsRaw` = 那次工具调用时模型实际传给工具的全部原始参数**（未经加工、原样保留）。构建代码：

```js
...node.call !== null ? { inputDetail: node.call.argsRaw } : {},   // payload（输入方向）
outputDetail: detailResult(node),                                    // 结果（输出方向）
```

UI 渲染逻辑：`direction === "input"` 显示 `inputDetail`（标签 "Payload"），`direction === "output"` 显示 `outputDetail`（标签 "Result"）。未捕获时显示 "No payload captured" / "No result captured"。

举例：本次会话中调用 `pwsh` 工具时，轨迹里那条 tool 记录的 payload 就是 `{"command": "Get-ChildItem ...", "description": "..."}` 这样的原始调用参数。工具结果若是合法 JSON，轨迹面板会用 JSON 树展示。

---

## 六、KV 缓存命中机制

### 6.1 本质：DeepSeek 服务端前缀缓存

高缓存命中**不是框架"主动命中"缓存，而是保证请求前缀稳定，让 DeepSeek 服务端的前缀缓存（prefix caching / KV cache）自动命中**。框架只做了一件事：把"每次都要重新算的部分"压缩到最小，把"不变的部分"保持字节级稳定。

适配器把官方返回的命中 token 映射为轨迹面板的 `cacheRead`：

```js
// dsh-llm-deepseek/lib/index.js
const cacheRead = usage.prompt_tokens_details?.cached_tokens ?? usage.prompt_cache_hit_tokens;
inputTokens: usage.prompt_tokens - (cacheRead ?? 0),  // 计费输入 = 总输入 - 缓存命中
```

注意：DeepSeek 的 `prompt_tokens` **包含**缓存命中部分，框架要把它减去才是真正的"新算力"输入。

### 6.2 缓存命中的主要内容（按体积排序）

1. **系统提示词**（最大头）：DSH 系统提示词 + 会话 persona/工作规则，每次逐字相同
2. **工具 schema**：当前挂载的全部工具定义，工具目录不变即完全重复
3. **历史消息前缀**：早先轮次的 user 消息、assistant 回复、工具调用/结果
4. **推理回传**（条件性）：仅含工具调用的 assistant 轮次才回传 `reasoning_content`

> **一句话：命中主体 = 系统提示词 + 工具目录 + 历史对话，即请求的"骨架部分"；每次新增的是本轮新消息和工具结果，追加在可复用前缀之后。**

### 6.3 DSH 做到高命中率的五条设计

1. **历史严格"只追加"（append-only）**：loop 保留的响应块只追加到下一个请求尾部，前面的 token 一个不动；表层替换或压缩会从第一个被遮蔽 token 起使复用失效。
2. **工具目录跨模式稳定**：计划模式与正常模式**不切换工具列表**（即使某工具计划模式下不可用也照样列出）——"The tool catalog stays the same across modes for request-cache stability."
3. **压缩时逐字回放前缀**：摘要调用**特意**逐字回放系统提示词、工具与已遮蔽区域消息，将压缩指令作为最后一条 user 消息追加——"复用提供方的热前缀 cache，而非使它失效"。
4. **排除噪声进历史**：只有表层事件进入模型历史；流分片、生命周期边界等仅写入日志的事件被排除。
5. **组装瀑布的字节级一致性**：系统提示词、schema 的组装是确定性的，不掺入会变化的文本（如时间戳单独注入，不进系统提示词）。

### 6.4 什么会击穿缓存

| 事件                              | 后果                              |
| --------------------------------- | --------------------------------- |
| 切换 provider / model             | 完全不同的缓存域，全部失效        |
| 修改系统提示词、工具 schema、前缀 | 从第一个变化的 token 起失效       |
| 压缩替换（checkpoint）            | 从被遮蔽的第一个历史 token 起失效 |
| 工具结果被剪枝改写                | 从该结果起失效                    |

**结论**：高命中率 = 框架把请求前缀做"稳定"，命中是"省出来的"而非"做出来的"。长会话中 cacheRead 动辄几十万 token 而 inputTokens（新增计算）只有几千，正是因为绝大部分请求内容是重复发送的历史前缀。

---

## 七、推理回传（Reasoning Passback）

### 7.1 定义

> **推理回传 = 思考模式（thinking mode）下，携带工具调用的 assistant 轮次必须把模型的思考过程（`reasoning_content`）一并写入消息历史，作为后续对话上下文。**

这是 DeepSeek 思考模式 API 的**硬性协议要求**（thinking-mode passback），不是可选优化。缺失时服务端返回 **400 错误**。多个主流框架都因此专门修复过：

- [[OpenAI Codex：DeepSeek thinking mode 缺失 reasoning_content 报 400](https://github.com/openai/codex/issues/24500)](https://github.com/openai/codex/issues/24500)
- [[langchain：assistant tool-call 必须包含 reasoning_content](https://github.com/langchain-ai/langchain/pull/35093)](https://github.com/langchain-ai/langchain/pull/35093)
- [[微软 agent-framework：跨工具调用循环保留并回放 reasoning_content](https://github.com/microsoft/agent-framework/issues/5538)](https://github.com/microsoft/agent-framework/issues/5538)
- [[Zed：修复 DeepSeek Reasoner 工具调用处理](https://github.com/zed-industries/zed/pull/44497)](https://github.com/zed-industries/zed/pull/44497)
- [[雷峰网：DeepSeek 文档更新，Agent 开发者要注意这个字段](https://www.leiphone.com/category/ai/3KPgkefKuQNtTxzN.html)](https://www.leiphone.com/category/ai/3KPgkefKuQNtTxzN.html)

### 7.2 DSH 适配器的实现规则

```js
// dsh-llm-deepseek/lib/index.js serializeAssistant()
return {
  role: "assistant",
  content: text,
  // 只有"有工具调用 且 有推理内容"时才回传 reasoning_content
  ...toolCalls.length > 0 && reasoning.length > 0 ? { reasoning_content: reasoning } : {},
  ...toolCalls.length > 0 ? { tool_calls: toolCalls } : {}
};
```

| 场景                            | 是否回传推理 | 原因                     |
| ------------------------------- | ------------ | ------------------------ |
| 有工具调用 + 有推理             | ✅ 回传       | 思考模式 API 强制要求    |
| 有工具调用 + 无推理（推理关闭） | ❌ 省略       | 没有可回传的内容         |
| 无工具调用（纯文本回复轮）      | ❌ 丢弃推理   | 终态不需要续接，省 token |

### 7.3 为什么会有这个机制

**① 协议硬约束**：思考模式下，任何带 `tool_calls` 的 assistant 消息必须携带 `reasoning_content`，否则 400。这是服务端对消息格式的强制校验。

**② 思考的"中断-延续"特性（深层原因）**：思考模式的推理 token 是自回归解码历史的一部分——模型"先思考、后回答"，思考与答案是同一段生成流。

- **单轮对话**：推理使命已完成，答案（`content`）就是思考的产出物，推理用完即弃，下一轮用户提问是新输入，模型从头思考即可。
- **工具调用**：模型发出 `tool_calls` 后对话被**中断**——它还没说完话，在等工具结果回来才能继续。此时推理记录的是"我为什么调用这个工具、我想确认什么"，是**接下来解读工具结果所必需的上下文**。工具结果返回后模型要**接着自己上一轮的想法继续思考**。推理不回传，模型就丢失了发起调用的动机链，解读结果的逻辑会断裂。

### 7.4 为什么普通步骤的推理无需回传

|                      | 普通回复轮                            | 工具调用轮                                 |
| -------------------- | ------------------------------------- | ------------------------------------------ |
| 该轮状态             | **终态**（话说完了）                  | **中断态**（等工具结果回来才能继续）       |
| 推理的作用           | 通往答案的思考，答案已在 `content` 里 | 发起调用的动机链，是后续解读工具结果的依据 |
| 后续是否依赖这段推理 | ❌ 不依赖，下轮是新输入                | ✅ 依赖，模型要"接上"自己的思考             |
| 省略推理的后果       | 无——答案完整，思考不参与续接          | 400 错误（协议）+ 思考断裂（语义）         |
| 回传的成本           | 白白膨胀历史、多付 token              | 必须支付，换取合法的多轮工具调用           |

**一句话概括**：

> 普通回复是"思考的终点"——推理使命已完成，产出已交付，下一轮从头思考即可，回传纯属浪费；工具调用是"思考的中断"——模型话没说完，推理必须保留下来，才能在工具结果回来后接着想，这是协议要求，也是思考连续性的需要。

### 7.5 与缓存的关系

- 推理回传在工具往返期间**追加**：它附加在请求前缀尾部（该条 assistant 消息内部），**不破坏**更早部分（系统提示词、历史）的缓存复用；
- 推理内容本身每次是新的（每轮思考不同），属于 cache miss，但它后面紧跟的 tool 结果等又是追加的；
- 无工具调用的轮次丢弃推理 → 历史更精简、前缀更稳定、不产生多余 token 成本。

---

## 八、总结

1. **插件化**：DSH 采用"最小核心 loop + 事件驱动插件"架构。loop 是唯一含具体循环逻辑的插件，其余一切策略（压缩、重试、权限、子代理、持久化、UI）都是挂在其事件体系上的外围插件；Agent 预设（标准/PTC/极简/创造）本质是插件组装方案。

2. **能力 seam 模式**：以压缩为例——接口（`dsh-compaction`）与实现（`dsh-compaction-basic`）分离，接口只规定"做什么"，实现决定"怎么做"，两者可独立替换；替换入口是 preset 的 `agent.cordis.yml`。

3. **缓存**：高命中率来自框架把请求前缀做"字节级稳定"（只追加历史、工具目录跨模式稳定、压缩回放前缀、排除噪声），命中主体是系统提示词 + 工具 schema + 历史消息前缀；命中由 DeepSeek 服务端前缀缓存自动完成，框架不主动"做"缓存，只是"省"出来的。

4. **推理回传**：思考模式下工具调用轮必须回传 `reasoning_content`（协议强制 + 思考连续性），普通回复轮丢弃推理（终态无需续接 + 省 token）；DSH 适配器精确执行这条规则，并以"丢弃"策略优化成本与缓存。

---

*文档基于 DeepSeek Harness 框架源码核实整理，主要来源：`dsh-agent-loop`、`dsh-compaction(-basic/-tool-result-pruner)`、`dsh-llm-deepseek`、`dsh-client-ui-trajectory`、`dsh/config/agent-presets/*` 等包的 README 与源码。*