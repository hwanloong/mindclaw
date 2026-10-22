<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,minimum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="color-scheme" content="dark">
<meta name="theme-color" content="#f2f3ef">
<title>终末地 · ENDFIELD TERMINAL</title>
<style>
  /* ── 色板：全部由 onTheme 覆写，默认值仅供浏览器预览 ───────────── */
  :root{
    --fs:1;
    --page:#FBFAF7; --text:#1B1B1A; --muted:#8A857C;
    --surface:#F1EFE9; --bubble:#E7EDE9; --primary:#2E4B45;
    --onPrimary:#FFFFFF;
    --hair:rgba(27,27,26,.10);
    --veil:rgba(27,27,26,.045);
    --shadow:0 1px 2px rgba(27,27,26,.05);
    --danger:#B4472F;
    --warn:#C8871B;
    /* 标记（荧光笔）底色：半透明，笔迹要透出底下的字 */
    --mk-yellow:rgba(255,224,130,.42);
    --mk-green:rgba(76,217,140,.38);
    --mk-blue:rgba(77,163,255,.36);
    --mk-pink:rgba(255,155,192,.40);
    --mk-gray:rgba(158,158,158,.34);
    --serif:"Iowan Old Style","Songti SC","Source Han Serif SC","Noto Serif CJK SC",Georgia,serif;
    --sans:-apple-system,BlinkMacSystemFont,"PingFang SC","Noto Sans CJK SC","Microsoft YaHei",sans-serif;
    --mono:"SF Mono",Menlo,Consolas,"Roboto Mono","Noto Sans Mono",monospace;
  }

  *{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
  html,body{height:100%;margin:0;overflow:hidden;overscroll-behavior:none}
  /* 换主题时底色软过渡，避免从深到浅整屏“啪”一下反白 */
  html,body{transition:background-color .18s ease,color .18s ease}

  /* html 也要上色：只给 body 上色时，键盘顶起 / 回弹的瞬间会露出
     WebView 默认白底 —— 这是"换背景色后发白"最直接的来源。 */
  html{background:var(--page)}

  body{
    display:flex;flex-direction:column;
    position:relative;
    background:var(--page);color:var(--text);
    font:calc(15.5px * var(--fs))/1.72 var(--sans);
    -webkit-font-smoothing:antialiased;
    text-rendering:optimizeLegibility;
  }
  /* 刻意**不叠任何渐变/蒙层**：宿主给什么底色就显示什么底色，
     多一层 tint 会让换色后整体发灰发白。 */

  /* 模型选择器并进输入框的下排：+ 图片 · 模型 · 发送（按手绘稿） */
  .toolrow{
    display:flex;align-items:center;gap:6px;
    padding:0 1px;
  }
  .toolrow .model{
    /* 不再是"占满剩余宽度"：模型名长短不一，撑满会把发送挤到边上、
       也显得笨重。按内容宽度走，上限 58%，超了省略。 */
    flex:0 1 auto;min-width:0;max-width:58%;
    text-align:left;
    overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
    padding:7px 14px;border-radius:18px;
    margin-right:auto;
    background:var(--veil);color:var(--muted);
    font:calc(12px * var(--fs))/1.4 var(--sans);
    letter-spacing:.02em;
    -webkit-tap-highlight-color:transparent;
    cursor:pointer;
  }
  .toolrow .model:active{background:var(--hair);color:var(--text)}

  /* ── 消息区 ──────────────────────────────────────────────────── */
  .stage{
    position:relative;z-index:1;flex:1 1 auto;min-height:0;
    display:flex;
  }
  .list{
    position:relative;flex:1 1 auto;min-width:0;
    overflow-y:auto;overflow-x:hidden;
    -webkit-overflow-scrolling:touch;
    overscroll-behavior:contain;
    padding:6px 20px 108px;
    display:flex;flex-direction:column;gap:22px;
  }

  /* ── 右缘滑动定位器：右滑呼出，拖到哪定位到哪 ──────────────── */
  .scrub{
    position:absolute;top:0;right:0;bottom:0;width:38px;z-index:6;
    opacity:0;pointer-events:none;transition:opacity .18s ease;
  }
  .scrub.on{opacity:1}
  .srail{
    position:absolute;right:7px;top:12px;bottom:12px;width:5px;
    border-radius:3px;background:var(--veil);
    box-shadow:inset 0 0 0 1px var(--hair);
  }
  .scrub.on .srail{pointer-events:auto;touch-action:none}
  .sthumb{
    position:absolute;left:-4px;width:13px;min-height:34px;
    border-radius:7px;background:var(--muted);opacity:.4;
    transition:opacity .15s ease;
  }
  .scrub.dragging .sthumb{opacity:1;background:var(--primary)}
  .stick{
    position:absolute;right:-2px;width:9px;height:2px;border-radius:1px;
    background:var(--text);opacity:.26;transition:opacity .15s ease,width .15s ease;
  }
  .scrub.dragging .stick{opacity:.5}
  .stick.cur{opacity:1;background:var(--primary);width:15px;right:-5px}
  /* 到顶/到底的接触反馈：不再是自己的橡皮筋（位移 + 弹簧回位），
     只在边缘浮一层很淡的光。内容一动不动，反馈靠"看见"。
     只写 opacity（合成层属性），不触发布局、不重建滚动瓦片。 */
  .edgeglow{
    position:absolute;left:0;right:0;height:92px;z-index:4;
    pointer-events:none;opacity:0;
    transition:opacity .2s ease-out;      /* 松手：慢慢化开 */
  }
  .edgeglow.top{top:0;background:linear-gradient(to bottom,var(--primary),transparent)}
  /* 底部这条贴在"正文真正能到的下沿"（输入框上沿），
     不然 92px 全被玻璃框盖住，等于没画。--composer-h 由 syncComposerPad 维护。 */
  .edgeglow.bottom{bottom:var(--composer-h,112px);background:linear-gradient(to top,var(--primary),transparent)}
  .edgeglow.live{transition:opacity .08s linear}   /* 跟手阶段：别拖尾 */
  .scard{
    position:absolute;right:30px;transform:translateY(-50%);
    max-width:230px;padding:7px 11px;border-radius:9px;
    background:var(--surface);border:1px solid var(--hair);
    box-shadow:0 8px 22px rgba(0,0,0,.20);
    font:calc(12px * var(--fs))/1.5 var(--sans);color:var(--text);
    opacity:0;transition:opacity .12s ease;pointer-events:none;
    white-space:nowrap;overflow:hidden;text-overflow:ellipsis;
  }
  .scrub.dragging .scard{opacity:1}

  /* ── 空态 ────────────────────────────────────────────────────── */
  .empty{
    margin:auto;text-align:center;padding:20px 8px;max-width:340px;
    animation:rise .5s cubic-bezier(.2,.7,.3,1) both;
  }
  .empty .glyph{
    width:54px;height:54px;margin:0 auto 20px;
    border:1.5px solid var(--primary);border-radius:3px;
    display:grid;place-items:center;color:var(--primary);
    font:400 26px/1 var(--serif);opacity:.9;transform:rotate(-2deg);
  }
  .empty h1{margin:0 0 6px;font:400 calc(21px * var(--fs))/1.5 var(--serif);letter-spacing:.06em}
  .empty p{margin:0;color:var(--muted);font-size:calc(13px * var(--fs))}

  /* ── 行 / 用户气泡 ───────────────────────────────────────────── */
  /* .row 必须是 flex 容器，.me 的 align-self 才有意义。
     contain:layout —— 一条消息内部重排不牵连整棵树。注意只能用
     layout：带上 style 会把行内 iframe 卡片整个画空（实测踩坑）。 */
  .row{
    display:flex;flex-direction:column;align-items:stretch;
    animation:rise .34s cubic-bezier(.2,.7,.3,1) both;
    contain:layout;
  }
  .me{
    align-self:flex-end;max-width:80%;min-width:0;
    background:var(--bubble);
    padding:10px 15px;
    border-radius:17px 17px 5px 17px;
    white-space:pre-wrap;word-break:break-word;
    font-size:calc(15px * var(--fs));line-height:1.66;
  }
  /* 已发送消息里的附件：图片给缩略图，文件给芯片。
     没有这一层的话，"带附件发出去"在界面上完全看不出来 ——
     重开应用后更是只剩一句正文。 */
  .meatts{
    display:flex;flex-wrap:wrap;gap:6px;
    white-space:normal;margin:0 0 8px;
  }
  .meatts:last-child{margin-bottom:0}
  .meimg{
    width:104px;height:104px;border-radius:11px;object-fit:cover;
    background:var(--hair);display:block;cursor:zoom-in;
    box-shadow:inset 0 0 0 1px rgba(0,0,0,.06);
  }
  .meimg.loading{opacity:.45}
  .mefile{
    display:inline-flex;align-items:center;gap:6px;max-width:196px;
    padding:5px 9px;border-radius:9px;
    background:var(--surface);
    box-shadow:inset 0 0 0 1px var(--hair),0 1px 4px rgba(0,0,0,.10);
    cursor:pointer;
  }
  .mefile .ficon{width:16px;height:16px;flex:0 0 auto;color:var(--muted)}
  .mefile b{
    font-weight:600;font-size:calc(11.5px * var(--fs));color:var(--text);
    opacity:.82;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
  }

  /* ── AI 消息 ─────────────────────────────────────────────────── */
  .ai{
    max-width:100%;padding-left:15px;
    border-left:1.5px solid var(--hair);margin-left:1px;
  }
  .ai.pending{border-left-color:var(--primary)}

  .meta{display:flex;flex-wrap:wrap;align-items:center;gap:6px;margin:0 0 8px}

  .tag{
    display:inline-flex;align-items:center;gap:5px;
    font:calc(11px * var(--fs))/1 var(--sans);
    letter-spacing:.04em;color:var(--muted);
    padding:5px 9px;border-radius:7px;
    background:var(--veil);border:1px solid transparent;
    -webkit-appearance:none;appearance:none;
  }
  .tag:active{background:var(--hair)}
  .tag .caret{
    width:0;height:0;border-left:4px solid currentColor;
    border-top:3px solid transparent;border-bottom:3px solid transparent;
    transition:transform .18s ease;opacity:.55;
  }
  .tag.open .caret{transform:rotate(90deg) translateX(-1px)}
  .tag.active{background:var(--hair);border-color:var(--hair);color:var(--text)}

  .dot{width:5px;height:5px;border-radius:50%;background:var(--muted);flex:0 0 auto}
  .dot.awaitingConfirm{background:var(--warn)}
  .dot.running{background:var(--primary);animation:pulse 1.1s ease-in-out infinite}
  .dot.success{background:var(--primary);opacity:.5}
  .dot.failed{background:var(--danger)}
  .dot.rejected{background:var(--muted);opacity:.38}
  /* 这一轮结束了就别再转：残留的 running 只是数据没刷新，不是真在跑 */
  .ai.idle .dot.running{animation:none;opacity:.45}
  .tag.tool.failed{color:var(--danger);background:rgba(180,71,47,.07)}
  .tag.tool .st{opacity:.55;font-size:.94em}
  .tag.tool .nm{letter-spacing:.02em}
  .tag.toolsum{color:var(--muted)}
  .tag.toolsum.has-failed{color:var(--danger);background:rgba(180,71,47,.07)}
  .tag.toolsum .n{opacity:.55;font-size:.94em}
  /* 折叠面板里的工具 chip 行 */
  .fold .tags{display:flex;flex-wrap:wrap;gap:5px;padding:9px 10px}
  .fold .tags + .fttl{border-top:1px solid var(--hair);border-bottom:0}

  .fold{
    margin:0 0 9px;
    border:1px solid var(--hair);border-radius:9px;
    background:var(--veil);overflow:hidden;
  }
  .fold[hidden]{display:none}
  .fold .fttl{
    display:flex;align-items:center;gap:6px;
    padding:6px 10px;border-bottom:1px solid var(--hair);
    font:calc(10.5px * var(--fs))/1 var(--sans);
    letter-spacing:.1em;text-transform:uppercase;color:var(--muted);
  }
  .fold pre{
    margin:0;padding:9px 11px;
    font:calc(11.5px * var(--fs))/1.6 var(--mono);
    color:var(--muted);white-space:pre-wrap;word-break:break-word;
    max-height:220px;overflow:auto;
  }
  .fold pre + pre{border-top:1px dashed var(--hair)}
  .reasoning pre{font-family:var(--sans);font-size:calc(12px * var(--fs));opacity:.9}

  /* ── 引用（消息自带的 quote）───────────────────────────────── */
  .quote{
    display:flex;gap:8px;margin:0 0 8px;
    padding:7px 10px;border-radius:8px;
    background:var(--veil);
    border-left:2.5px solid var(--muted);
    font:calc(12.5px * var(--fs))/1.55 var(--sans);
    color:var(--muted);max-width:100%;
  }
  .quote.mine{align-self:flex-end;max-width:80%}
  .quote .qt{
    min-width:0;display:-webkit-box;-webkit-box-orient:vertical;
    -webkit-line-clamp:4;overflow:hidden;white-space:pre-wrap;word-break:break-word;
  }

  /* ── 文字标记（荧光笔 / 引用虚线）────────────────────────────── */
  .mk{
    padding:.05em .08em;border-radius:3px;
    -webkit-box-decoration-break:clone;box-decoration-break:clone;
  }
  .mk.yellow{background:var(--mk-yellow)}
  .mk.green{background:var(--mk-green)}
  .mk.blue{background:var(--mk-blue)}
  .mk.pink{background:var(--mk-pink)}
  .mk.gray{background:var(--mk-gray)}
  .mk.q{background:none;border-bottom:1.5px dashed currentColor;padding:0}
  .mk.q.yellow{color:#D9A400}
  .mk.q.green{color:#2F9E5C}
  .mk.q.blue{color:#3277C9}
  .mk.q.pink{color:#D95C86}
  .mk.q.gray{color:#8A8A8A}

  /* ── Markdown 正文 ──────────────────────────────────────────── */
  .body{
    font-family:var(--serif);
    font-size:calc(15.5px * var(--fs));
    line-height:1.78;word-break:break-word;letter-spacing:.01em;
  }
  .body > *:first-child{margin-top:0}
  .body > *:last-child{margin-bottom:0}
  .body p{margin:0 0 .82em;white-space:pre-wrap}
  .body h1,.body h2,.body h3,.body h4{
    font-weight:600;line-height:1.45;letter-spacing:.02em;margin:1.35em 0 .5em;
  }
  .body h1{font-size:1.32em}
  .body h2{font-size:1.18em}
  .body h3{font-size:1.05em}
  .body h4{font-size:.98em;color:var(--muted)}
  .body strong{font-weight:600}
  .body em{font-style:italic;color:var(--muted)}
  .body s{opacity:.5}
  .body a{color:var(--primary);text-decoration:none;border-bottom:1px solid var(--hair)}
  .body ul,.body ol{margin:0 0 .82em;padding-left:1.35em}
  .body li{margin:.18em 0}
  .body li::marker{color:var(--muted)}
  .body blockquote{
    margin:0 0 .82em;padding:2px 0 2px 13px;
    border-left:2px solid var(--hair);color:var(--muted);
  }
  .body blockquote p:last-child{margin-bottom:0}
  .body hr{border:0;border-top:1px solid var(--hair);margin:1.4em 0}
  .body code{
    font-family:var(--mono);font-size:.855em;
    background:var(--veil);border-radius:4px;padding:.14em .38em;
  }
  .body .tablewrap{overflow-x:auto;margin:0 0 .9em}
  .body table{border-collapse:collapse;font-family:var(--sans);font-size:.86em;min-width:100%}
  .body th,.body td{border:1px solid var(--hair);padding:6px 10px;text-align:left;white-space:nowrap}
  .body th{background:var(--veil);font-weight:600}

  /* ── 代码卡片 ───────────────────────────────────────────────── */
  .codecard{
    margin:0 0 .9em;border:1px solid var(--hair);border-radius:10px;
    background:var(--veil);overflow:hidden;
  }
  .codecard .cap{display:flex;align-items:center;gap:6px;padding:5px 6px 5px 11px}
  .codecard .lang{
    font:calc(10.5px * var(--fs))/1 var(--mono);
    letter-spacing:.1em;text-transform:uppercase;color:var(--muted);
  }
  .codecard .grow{flex:1 1 auto}
  .mini{
    border:1px solid transparent;background:transparent;color:var(--muted);
    font:calc(11px * var(--fs))/1 var(--sans);
    padding:5px 8px;border-radius:6px;letter-spacing:.04em;
  }
  .mini:active{background:var(--hair);color:var(--text)}
  .mini.done{color:var(--primary)}
  .codecard pre{
    margin:0;padding:10px 12px 12px;border:0;border-radius:0;
    background:transparent;overflow-x:auto;
  }
  .codecard pre code{
    background:none;padding:0;display:block;white-space:pre;
    font-size:calc(12.5px * var(--fs));line-height:1.62;
  }
  .codecard .pv{border-top:1px solid var(--hair);background:var(--surface)}
  .codecard .pv[hidden]{display:none}
  .codecard .pv iframe{
    display:block;width:100%;height:190px;border:0;background:transparent;
    /* 新 iframe 第一帧是白的 —— 载入完成才显形，切主题/重建都不闪白 */
    opacity:0;transition:opacity .18s ease;
  }
  .codecard .pv iframe.rdy{opacity:1}
  .codecard .pv .pbar{
    display:flex;align-items:center;gap:6px;padding:5px 11px;
    font:calc(10px * var(--fs))/1 var(--sans);letter-spacing:.1em;
    text-transform:uppercase;color:#8A857C;border-bottom:1px solid #eee;
  }

  /* HTML / SVG / mermaid / 图片 / 交互卡：默认显示渲染结果，源码收起来。
     JS 卡例外 —— 源码必须一直可见（用户要求），运行结果另起一块。 */
  .codecard[data-kind="html"]:not(.showsrc) > pre,
  .codecard[data-kind="map"]:not(.showsrc) > pre,
  .codecard[data-kind="image"]:not(.showsrc) > pre,
  .codecard[data-kind="mermaid"]:not(.showsrc) > pre,
  .codecard[data-kind="htmlcard"]:not(.showsrc) > pre{display:none}
  .codecard[data-kind="html"].showsrc .pv,
  .codecard[data-kind="map"].showsrc .map,
  .codecard[data-kind="image"].showsrc .pv,
  .codecard[data-kind="mermaid"].showsrc .pv,
  .codecard[data-kind="htmlcard"].showsrc .pv{display:none}
  .codecard .pv.live iframe{
    display:block;width:100%;height:120px;border:0;background:transparent;
  }
  .codecard .imgc{padding:8px;text-align:center}
  .codecard .imgc img{
    max-width:100%;max-height:300px;border-radius:8px;
    border:1px solid var(--hair);background:var(--veil);
  }
  .nofs2{
    display:inline-block;padding:8px 12px;border-radius:8px;
    background:var(--veil);border:1px solid var(--hair);
    font:calc(12px * var(--fs))/1.5 var(--sans);color:var(--muted);
  }
  .codecard .jsout iframe{display:none}

  /* 思维导图 */
  .codecard .map{
    overflow-x:auto;overflow-y:hidden;
    border-top:1px solid var(--hair);padding:10px 6px;
    -webkit-overflow-scrolling:touch;
  }
  .codecard .map svg{display:block;max-width:none}
  .mn-rect{fill:var(--veil);stroke:var(--hair);stroke-width:1}
  .mn-root{fill:var(--primary);stroke:none}
  .mn-t{fill:var(--text);font:calc(12px * var(--fs))/1 var(--sans)}
  .mn-t-root{fill:var(--onPrimary);font-weight:600}
  .mn-link{fill:none;stroke:var(--hair);stroke-width:1.4}

  /* JS 卡片：运行输出 */
  .codecard .jsout{
    display:block;border-top:1px solid var(--hair);
    padding:8px 11px;max-height:186px;overflow:auto;
    background:var(--veil);
    font:calc(11.5px * var(--fs))/1.62 var(--mono);
    color:var(--text);white-space:pre-wrap;word-break:break-word;
  }
  .codecard .jsout[hidden]{display:none}
  .codecard .jsout .ln{display:block}
  .codecard .jsout .ln.err{color:var(--danger)}
  .codecard .jsout .ln.dim{opacity:.55}
  /* 运行用的 iframe 不能露出来：它只是个执行环境，输出走 .jsout。
     注意别用 display:none —— 部分 Chromium/WebView 版本对
     "不参与布局"的 iframe 会推迟甚至跳过 srcdoc 执行，表现就是
     "点了运行、卡片里一行输出都没有"。移出视口更保险。 */
  iframe.jsrun{
    position:absolute;left:-10000px;top:0;
    width:2px;height:2px;border:0;opacity:0;pointer-events:none;
  }

  /* 卡片内的提示/报错条：代替原生 alert，贴在卡片底部 */
  .codecard .cnote{
    display:block;padding:6px 11px 8px;
    border-top:1px dashed var(--hair);
    font:calc(11.5px * var(--fs))/1.55 var(--mono);
    color:var(--muted);white-space:pre-wrap;word-break:break-word;
  }
  .codecard .cnote[hidden]{display:none}
  .codecard .cnote .cnrow{display:block}
  .codecard .cnote .cnrow.bad{color:var(--danger)}

  .plain{white-space:pre-wrap;word-break:break-word;font-family:var(--sans)}

  .caretline{
    display:inline-block;width:2px;height:1em;
    background:var(--primary);vertical-align:-2px;
    margin-left:2px;animation:blink 1s steps(2,start) infinite;
  }
  .thinking{display:inline-flex;gap:4px;align-items:center;padding:4px 0}
  .thinking i{
    width:5px;height:5px;border-radius:50%;background:var(--primary);
    opacity:.3;animation:bounce 1.25s ease-in-out infinite;
  }
  .thinking i:nth-child(2){animation-delay:.16s}
  .thinking i:nth-child(3){animation-delay:.32s}

  .err{
    margin:9px 0 0;padding:8px 11px;border-radius:8px;
    background:rgba(180,71,47,.07);color:var(--danger);
    font:calc(12.5px * var(--fs))/1.55 var(--sans);white-space:pre-wrap;
  }

  /* ── 输入区：没有底板，只有输入框本身是模糊玻璃 ────────────── */
  .composer{
    /* fixed 而不是 absolute：跟着布局视口走，键盘收起时不需要等
       JS 改 body 高度才落下来 —— 那半拍延迟就是"卡一下才落下"。 */
    position:fixed;left:0;right:0;bottom:0;z-index:5;
    /* 底部留够：贴太死会被手势条压住；键盘弹出时它还得浮在输入法上面。
       22px 是"看着不贴边、又不浪费正文高度"的位置（原来 14px 顶在手势条上）。 */
    padding:0 14px calc(22px + env(safe-area-inset-bottom));
    /* 刻意不加任何背景/模糊/描边：正文从这一层下面穿过去，
       只有 .box 那一条玻璃会把背后的字糊掉。 */
    background:transparent;
    pointer-events:none;
    will-change:transform;
  }
  /* 底板没了，这一层的点击要重新交还给子元素 */
  .composer > *{pointer-events:auto}

  /* 待发引用条：挂在输入框上面，可删 */
  .qbar{
    display:flex;align-items:flex-start;gap:8px;
    max-height:0;opacity:0;overflow:hidden;margin:0 4px;padding:0 10px;
    border-left:3px solid var(--qc,#D9A441);
    background:var(--veil);border-radius:0 9px 9px 0;
    transition:max-height .22s ease,opacity .22s ease,padding .22s ease,margin .22s ease;
  }
  .qbar.on{max-height:110px;opacity:1;padding:8px 10px;margin:0 4px 7px}
  .qbar .qt{
    flex:1;min-width:0;color:var(--text);opacity:.82;
    font:calc(12.5px * var(--fs))/1.55 var(--sans);
    display:-webkit-box;-webkit-line-clamp:3;-webkit-box-orient:vertical;overflow:hidden;
    white-space:pre-wrap;word-break:break-word;
  }
  .qbar .qx{
    flex:0 0 auto;width:24px;height:24px;border-radius:50%;border:0;padding:0;
    background:var(--hair);color:var(--text);opacity:.65;
    font:15px/1 var(--sans);display:grid;place-items:center;
  }
  .qbar .qx:active{opacity:1}

  /* 拖拽跳转的浮动提示 */
  .jhint{
    position:absolute;right:2px;top:50%;transform:translateY(-50%) scale(.8);
    padding:3px 8px;border-radius:7px;white-space:nowrap;
    background:var(--primary);color:var(--onPrimary);
    font:calc(11px * var(--fs))/1 var(--sans);letter-spacing:.06em;
    opacity:0;pointer-events:none;transition:opacity .12s ease,transform .12s ease;
  }
  .jump.dragging .jhint{opacity:1;transform:translateY(-50%) scale(1)}

  /* 回到底部 */
  .jump{
    position:absolute;right:20px;top:-44px;
    width:36px;height:36px;border-radius:50%;
    border:1px solid var(--hair);background:var(--surface);color:var(--text);
    display:grid;place-items:center;padding:0;
    box-shadow:0 2px 8px rgba(0,0,0,.12);
    opacity:0;transform:translateY(8px) scale(.86);pointer-events:none;
    transition:opacity .18s ease,transform .18s ease;
  }
  .jump.on{opacity:1;transform:none;pointer-events:auto}
  .jump svg{width:16px;height:16px;display:block}
  .jump{touch-action:none;user-select:none;-webkit-user-select:none}
  .jump .badge{
    position:absolute;top:-2px;right:-2px;
    width:9px;height:9px;border-radius:50%;
    background:var(--primary);border:2px solid var(--surface);
    opacity:0;transform:scale(.6);transition:opacity .2s ease,transform .2s ease;
  }
  .jump.dirty .badge{opacity:1;transform:none;animation:pulse 1.4s ease-in-out infinite}

  /* 附件条 */
  .attachbar{
    display:flex;align-items:center;gap:6px;flex-wrap:wrap;
    font:calc(11.5px * var(--fs))/1 var(--sans);
    color:var(--muted);
    max-height:0;overflow:hidden;opacity:0;padding:0 4px;
    transition:max-height .2s ease,opacity .2s ease,padding .2s ease;
  }
  .attachbar.on{max-height:96px;opacity:1;padding:6px 4px 2px}
  .atts{display:flex;align-items:center;gap:6px;flex-wrap:wrap;min-width:0}
  .att{
    display:inline-flex;align-items:center;gap:6px;max-width:200px;
    padding:4px 8px;border-radius:8px;
    /* 不透明底：附件芯片压在正文上，半透明会跟底下的字混在一起 */
    background:var(--surface);
    box-shadow:inset 0 0 0 1px var(--hair),0 1px 4px rgba(0,0,0,.10);
  }
  .att img{
    width:26px;height:26px;border-radius:5px;object-fit:cover;flex:0 0 auto;
    background:var(--hair);cursor:zoom-in;
  }
  .att .ficon{width:18px;height:18px;flex:0 0 auto;color:var(--muted)}
  .att b{
    font-weight:600;color:var(--text);opacity:.82;font-size:calc(11.5px * var(--fs));
    overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
  }
  .att .attx{
    flex:0 0 auto;width:17px;height:17px;border-radius:50%;border:0;padding:0;
    background:var(--hair);color:var(--text);opacity:.6;
    font:12px/1 var(--sans);display:grid;place-items:center;
  }
  .att .attx:active{opacity:1}
  .att[data-img="1"] b{cursor:zoom-in}
  .attachbar .hint{opacity:.6;font-size:calc(10.5px * var(--fs))}

  /* ── 数学公式 ────────────────────────────────────────────────── */
  .math{display:inline-block;white-space:normal;vertical-align:middle}
  .math.blk{display:block;text-align:center;overflow-x:auto;overflow-y:hidden;padding:6px 2px}
  .math .mi{font-style:italic}            /* 变量 */
  .math .mo{padding:0 .14em}              /* 运算符 */
  .math .mnu{font-style:normal}           /* 数字/函数名 */
  .math .mtx{font-style:normal;opacity:.9;padding:0 .18em}  /* \text */
  .math .frac{display:inline-flex;flex-direction:column;align-items:center;vertical-align:middle;margin:0 .18em}
  .math .frac>span{display:block;padding:0 .3em;line-height:1.25}
  .math .frac>span+span{border-top:1px solid currentColor}
  .math .sup,.math .sub{font-size:.72em;line-height:1}
  .math .sup{vertical-align:.55em}
  .math .sub{vertical-align:-.28em}
  .math .lim{display:inline-flex;flex-direction:column;align-items:center;vertical-align:middle;margin:0 .15em}
  .math .lim .bigo{font-size:1.35em;line-height:1;font-style:normal}
  .math .lim .ud{display:flex;flex-direction:column;align-items:center;font-size:.68em;line-height:1.25}
  .math .rt{display:inline-flex;align-items:stretch;vertical-align:middle;margin:0 .12em}
  .math .rt .sym{transform:translateY(.1em);font-style:normal}
  .math .rt .rad{border-top:1px solid currentColor;padding:0 .22em;margin-left:-.06em}
  .math .mendisp{display:inline-flex;flex-direction:column;align-items:flex-start;vertical-align:middle;margin:0 .1em}
  .math .mendisp .mrow{display:block;padding:.14em .3em;white-space:nowrap}

  /* @ 工具/插件选择器 */
  .mention{
    position:absolute;left:14px;right:14px;
    bottom:calc(100% - 4px);
    max-height:218px;overflow-y:auto;
    background:var(--surface);
    border:1px solid var(--hair);border-radius:12px;
    box-shadow:0 -8px 24px rgba(0,0,0,.16);
    padding:4px;z-index:8;
    animation:pop .14s cubic-bezier(.2,.8,.3,1) both;
  }
  .mention[hidden]{display:none}
  .mitem{
    display:flex;align-items:center;gap:9px;width:100%;
    text-align:left;border:0;background:transparent;
    padding:8px 9px;border-radius:8px;color:var(--text);
  }
  .mitem .ic{
    width:24px;height:24px;border-radius:7px;flex:0 0 auto;
    background:var(--veil);display:grid;place-items:center;
    font:600 calc(10.5px * var(--fs))/1 var(--sans);color:var(--primary);
  }
  .mitem .tx{min-width:0;flex:1 1 auto}
  .mitem .tt{
    font:600 calc(13px * var(--fs))/1.35 var(--sans);
    white-space:nowrap;overflow:hidden;text-overflow:ellipsis;
  }
  .mitem .nt{
    font:calc(11px * var(--fs))/1.4 var(--sans);color:var(--muted);
    white-space:nowrap;overflow:hidden;text-overflow:ellipsis;
  }
  .mitem.cur{background:var(--hair)}
  .mitem:active{background:var(--hair)}
  .mempty{
    padding:12px;text-align:center;
    font:calc(12px * var(--fs))/1.5 var(--sans);color:var(--muted);
  }

  /* 输入框：上排是输入内容，下排 + · 图片 · 模型 · 发送（按手绘稿）。
     "玻璃感"这次用真的 backdrop-filter —— 之前移除是因为整屏滚动
     每帧重模糊；现在模糊只作用于底部这一条玻璃（不是全屏），
     成本可控，换来的通透感是高不透明底给不了的。 */
  .box{
    display:flex;flex-direction:column;gap:3px;
    background:var(--surface);
    background:var(--glass-bg, var(--surface));
    border:1px solid var(--glass-line, var(--hair));
    border-radius:24px;
    padding:5px 6px 6px;
    box-shadow:0 6px 22px -8px rgba(0,0,0,.24),0 1px 2px rgba(0,0,0,.10);
    -webkit-backdrop-filter:blur(16px) saturate(150%);
    backdrop-filter:blur(16px) saturate(150%);
    transition:border-color .18s ease,background .18s ease;
  }
  .box:focus-within{border-color:var(--primary)}

  /* 滚动期间临时关掉模糊。
     backdrop-filter 是**每帧重算**的：列表在滚，玻璃下面的内容一直在变，
     WebView 每帧都要重新做一次 16px 高斯模糊 —— 这是滚动掉帧最大的一处。
     滚动时人的视线在正文上，底部这条几乎不看，所以用近似实色顶 180ms，
     停下再淡回模糊，观感几乎无感、帧率立刻回来。 */
  body.scrolling .box,body.kb .box{
    -webkit-backdrop-filter:none;
    backdrop-filter:none;
    background:var(--surface);
    transition:background .1s ease;
  }
  /* 键盘升降（body.kb）同样关掉模糊：输入法动画那 200~300ms 里，
     玻璃下面的内容一直在被顶动，每帧一次 16px 高斯模糊直接把动画拖垮。 */

  .icon{
    flex:0 0 auto;width:34px;height:34px;
    display:grid;place-items:center;
    border:0;border-radius:50%;background:transparent;
    color:var(--muted);padding:0;margin:0;
  }
  .icon:active{background:var(--hair);color:var(--text)}
  .icon svg{width:18px;height:18px;display:block}

  textarea#i{
    flex:1 1 auto;min-width:0;
    border:0;outline:0;background:transparent;color:var(--text);
    font:calc(15px * var(--fs))/22px var(--sans);
    height:36px;min-height:36px;
    padding:7px 8px 1px;margin:0;resize:none;
    caret-color:var(--primary);
  }
  textarea#i::placeholder{color:var(--muted);opacity:.72}

  .send{
    flex:0 0 auto;width:34px;height:34px;
    border:0;border-radius:50%;
    background:var(--primary);color:var(--onPrimary);
    display:grid;place-items:center;padding:0;
    transition:opacity .16s ease,transform .16s ease;
  }
  .send:disabled{opacity:.28}
  .send:active:not(:disabled){transform:scale(.93)}
  .send svg{width:17px;height:17px;display:block}
  .send.stop{background:var(--text);color:var(--page)}
  .send.stop svg{width:12px;height:12px}

  /* ── 图片 / 链接 / 文件卡片（正文里自动出现）───────────────── */
  .mdimg{
    display:block;margin:2px 0 .7em;max-width:100%;
  }
  .mdimg img{
    display:block;max-width:100%;max-height:260px;
    border-radius:10px;border:1px solid var(--hair);
    background:var(--veil);
  }
  .mdimg .mdcap{
    display:block;margin-top:4px;
    font:calc(11.5px * var(--fs))/1.5 var(--sans);color:var(--muted);
  }
  .mdimg .nofs{
    display:inline-block;padding:6px 10px;border-radius:8px;
    background:var(--veil);border:1px solid var(--hair);
    font:calc(12px * var(--fs))/1.5 var(--sans);color:var(--muted);
  }
  .lchip,.fchip{
    display:inline-flex;align-items:center;gap:4px;
    max-width:100%;padding:1px 7px;margin:0 2px;
    border-radius:6px;background:var(--veil);
    border:1px solid var(--hair);
    font:calc(11.5px * var(--fs))/1.7 var(--sans);
    color:var(--primary);vertical-align:baseline;
    white-space:nowrap;overflow:hidden;text-overflow:ellipsis;
  }
  .fchip{color:var(--text)}
  .lchip .tagx,.fchip .tagx{
    font-size:.86em;letter-spacing:.04em;color:var(--muted);opacity:.85;
  }

  /* 点开图片全屏 */
  .zoom{
    position:fixed;inset:0;z-index:40;
    background:rgba(0,0,0,.86);
    display:flex;flex-direction:column;align-items:center;justify-content:center;
    gap:10px;padding:16px;
    animation:fadein .16s ease both;
  }
  .zoom img{max-width:92vw;max-height:82vh;border-radius:8px;object-fit:contain}
  .zoom .cap{color:#d8d5cf;font:12px/1.5 var(--sans);text-align:center;max-width:90vw}
  @keyframes fadein{from{opacity:0}to{opacity:1}}

  /* 本地文件内容预览：宿主代读（白名单 + 4MB），这里只负责摊开看 */
  .fview{
    position:fixed;inset:0;z-index:41;
    background:var(--page);color:var(--text);
    display:flex;flex-direction:column;
    animation:fadein .16s ease both;
  }
  .fview .fhead{
    display:flex;align-items:center;gap:7px;
    padding:calc(10px + env(safe-area-inset-top)) 12px 10px;
    border-bottom:1px solid var(--hair);
  }
  .fview .fname{
    flex:1;min-width:0;
    font:calc(13.5px * var(--fs))/1.4 var(--sans);font-weight:600;
    white-space:nowrap;overflow:hidden;text-overflow:ellipsis;
  }
  .fview .fmeta{
    flex:none;color:var(--muted);
    font:calc(11px * var(--fs))/1.3 var(--sans);
  }
  .fview .fhead button{
    flex:none;border:1px solid var(--hair);background:var(--surface);color:var(--text);
    font:calc(12px * var(--fs))/1 var(--sans);
    padding:7px 10px;border-radius:9px;white-space:nowrap;
  }
  .fview .fhead button:active{background:var(--hair)}
  .fview .fbody{
    flex:1;overflow:auto;-webkit-overflow-scrolling:touch;
    padding:12px 14px calc(20px + env(safe-area-inset-bottom));
  }
  .fview pre{
    margin:0;white-space:pre-wrap;word-break:break-word;
    font:calc(12.5px * var(--fs))/1.68 var(--mono);
  }
  .fview .fnote{
    margin:0;color:var(--muted);
    font:calc(12.5px * var(--fs))/1.7 var(--sans);
  }

  /* 长按菜单 */
  .cmenu{
    position:fixed;z-index:31;left:0;top:0;
    display:flex;gap:1px;padding:5px;
    background:var(--surface);
    border:1px solid var(--hair);border-radius:11px;
    box-shadow:0 10px 26px rgba(0,0,0,.20);
    animation:pop .14s cubic-bezier(.2,.8,.3,1) both;
  }
  .cmenu button{
    border:0;background:transparent;color:var(--text);
    font:calc(12.5px * var(--fs))/1 var(--sans);
    padding:9px 11px;border-radius:7px;white-space:nowrap;
  }
  .cmenu button:active{background:var(--hair)}
  .cmenu button.danger{color:var(--danger)}
  @keyframes pop{from{opacity:0;transform:scale(.94)}to{opacity:1;transform:none}}

  /* ── 动效 ────────────────────────────────────────────────────── */
  @keyframes rise{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:none}}
  @keyframes blink{0%,100%{opacity:1}50%{opacity:0}}
  @keyframes pulse{0%,100%{opacity:1;transform:scale(1)}50%{opacity:.32;transform:scale(.82)}}
  @keyframes bounce{0%,60%,100%{transform:translateY(0);opacity:.28}30%{transform:translateY(-3px);opacity:.9}}
  @media (prefers-reduced-motion:reduce){
    *{animation-duration:.001s !important;transition-duration:.001s !important}
  }

  /* ── 深色 ────────────────────────────────────────────────────── */
  /* onTheme 到达之前先按系统深浅色出图，避免深色下闪一下白底 */
  @media (prefers-color-scheme: dark){
    :root{
      --page:#101211; --text:#E7E5DF; --muted:#8B8880;
      --surface:#1A1C1B; --bubble:#24322D; --primary:#6FB3A4;
      --onPrimary:#092420;
      --hair:rgba(231,229,223,.15);
      --veil:rgba(231,229,223,.06);
      --shadow:0 1px 2px rgba(0,0,0,.30);
    }
  }
  html[data-dark="1"]{
    --danger:#E0846A;
    --warn:#D8A94E;
    --shadow:0 1px 2px rgba(0,0,0,.30);
    --mk-yellow:rgba(255,213,79,.34);
    --mk-green:rgba(56,214,121,.32);
    --mk-blue:rgba(59,155,255,.32);
    --mk-pink:rgba(255,111,165,.32);
    --mk-gray:rgba(189,189,189,.30);
  }
  html[data-dark="1"] .err{background:rgba(224,132,106,.12)}
  html[data-dark="1"] .tag.tool.failed{background:rgba(224,132,106,.13)}
  html[data-dark="1"] .jump{box-shadow:0 2px 10px rgba(0,0,0,.42)}
  html[data-dark="1"] .codecard .pv{border-top-color:rgba(231,229,223,.14);background:#141615}
  html[data-dark="1"] .codecard .pv .pbar{
    color:#8B8880;background:#1A1C1B;border-bottom-color:rgba(231,229,223,.12);
  }
  html[data-dark="1"] .body a{border-bottom-color:rgba(111,179,164,.35)}

  /* ══ 终末地 · 冷黑工业编辑风 ══════════════════════════════════════
     令牌取自 dph-endfield-theme：深黑场（#0a0d0e）、纸白铬（#f2f3ef）、
     荧光信号黄（#f2f500）。全直角、切角面板、单像素细线、等宽小标签。
     参照系不是"卡片 UI"，是一块工业仪表盘：信息用线切块，不用阴影堆。 */
  :root{
    --dph-field:#0a0d0e;          /* 画布 */
    --dph-surface:#151a1b;        /* 面板 */
    --dph-surface-strong:#202728; /* 抬升层 / 用户气泡 */
    --dph-ink:#eef3ef;
    --dph-muted:#89938f;
    --dph-rule:rgba(225,235,232,.20);
    --dph-rule-soft:rgba(225,235,232,.10);
    --dph-rule-strong:rgba(225,235,232,.42);
    --dph-signal:#f2f500;
    --dph-signal-soft:rgba(215,242,47,.14);
    --dph-signal-line:rgba(215,242,47,.22);
    --dph-paper:#f2f3ef;
    --dph-ink-dark:#151717;
    --dph-safe:#72d6b5;
    --dph-warn:#f0bc4d;
    --dph-error:#ef655d;
    --dph-cut:polygon(0 0,calc(100% - 14px) 0,100% 14px,100% 100%,14px 100%,0 calc(100% - 14px));
    --dph-cut-sm:polygon(0 0,calc(100% - 8px) 0,100% 8px,100% 100%,8px 100%,0 calc(100% - 8px));

    /* 宿主 onTheme 仍会覆写这几项，同名映射保持兼容 */
    --page:var(--dph-field); --text:var(--dph-ink); --muted:var(--dph-muted);
    --surface:var(--dph-surface); --bubble:var(--dph-surface-strong);
    --primary:var(--dph-signal); --onPrimary:var(--dph-ink-dark);
    --hair:var(--dph-rule); --veil:rgba(225,235,232,.05);
    --danger:var(--dph-error); --warn:var(--dph-warn);
    --serif:var(--sans);
    --sans:Bahnschrift,"Arial Narrow","Segoe UI","PingFang SC","Noto Sans CJK SC","Microsoft YaHei",sans-serif;
    --mono:"SF Mono",Consolas,"Roboto Mono","Noto Sans Mono",monospace;
  }
  html{background:var(--dph-field)}
  body{
    background:var(--dph-field);color:var(--dph-ink);
    font-family:var(--sans);letter-spacing:.012em;
    font-variant-numeric:tabular-nums;
    /* 基础样式开了 optimizeLegibility（字体整形 + 连字），中文长文基本只付代价
       不收好处；正文一律回 auto，滚动时的字形整形开销直接省掉。 */
    text-rendering:auto;
  }

  /* 背景层：网格 + 矩阵色块 + 信号辉光，全部并成**一层**。
     原来拆成 ::before/::after 两个全屏层，还带一个 8s 无限旋转动画 ——
     常驻动画在手机上就是白烧电，两层全屏图层也多占一份纹理内存。
     现在只留一层静态底，漂移动效挪去启动动画里（那边本来就是一次性的）。 */
  body::before{
    content:"";position:fixed;inset:0;z-index:0;pointer-events:none;
    background:
      repeating-linear-gradient(90deg,rgba(225,235,232,.045) 0 1px,transparent 1px 96px),
      repeating-linear-gradient(0deg,rgba(225,235,232,.035) 0 1px,transparent 1px 96px),
      linear-gradient(90deg,transparent 0 19%,rgba(255,255,255,.028) 19% 32%,transparent 32% 61%,rgba(242,245,0,.05) 61% 69%,transparent 69%),
      linear-gradient(0deg,transparent 0 16%,rgba(255,255,255,.024) 16% 29%,transparent 29% 58%,rgba(255,255,255,.015) 58% 78%,transparent 78%),
      radial-gradient(circle at 82% 12%,rgba(242,245,0,.05),transparent 42%),
      linear-gradient(180deg,rgba(255,255,255,.022),transparent 30%);
  }
  /* 整页 1px 信号描边：仪表盘的外框 */
  .frame{
    position:fixed;inset:0;z-index:14;pointer-events:none;
    border:1px solid var(--dph-signal-line);
    border-top:0;
    box-shadow:inset 0 -1px 0 rgba(225,235,232,.10);
  }
  .stage,.composer{position:relative;z-index:1}

  /* ── 顶栏：纸白铬 ─────────────────────────────────────────────── */
  /* 高度统一由 --tbh 控制：正文区的顶部内边距、顶栏下方的分隔线都跟着它走，
     想再调高低只改这一个数。 */
  :root{--tbh:68px}
  .topbar{
    position:fixed;left:0;right:0;top:0;z-index:5;
    height:calc(var(--tbh) + var(--st,0px));
    padding:var(--st,0px) 14px 0;
    display:flex;align-items:center;gap:9px;
    background:var(--dph-paper);color:var(--dph-ink-dark);
    border:0;border-bottom:1px solid var(--dph-ink-dark);
    border-radius:0;
    box-shadow:0 10px 24px rgba(0,0,0,.28);
  }
  /* 顶栏入场等启动动画退场后再演一遍：不然它在盖子底下就播完了，
     擦除上来时看到的是一块静止的铬，白瞎了这段动效。 */
  html[data-booted="1"] .topbar{animation:dph-header-enter .52s cubic-bezier(.2,.8,.2,1) both}
  /* 顶栏不再压一条黑黄信号条：那条黑边贴在状态栏下方，看着像多出来的一行黑。
     信号黄改由左侧方块、状态章和输入框角标承担。 */
  .topbar .sq{
    width:15px;height:15px;flex:0 0 auto;border-radius:0;
    background:var(--dph-signal);border:1px solid var(--dph-ink-dark);
    box-shadow:0 0 0 3px rgba(242,245,0,.16);
  }
  .topbar b{
    font:800 calc(20px * var(--fs))/1 var(--sans);
    letter-spacing:.06em;color:var(--dph-ink-dark);
    white-space:nowrap;text-shadow:none;
  }
  .topbar .sub{
    font:700 calc(9px * var(--fs))/1.3 var(--mono);
    letter-spacing:.18em;text-transform:uppercase;
    color:#555b59;white-space:nowrap;
    border-left:1px solid rgba(21,23,23,.28);padding-left:9px;
    opacity:1;
  }
  .topbar .rt{
    margin-left:auto;display:inline-flex;align-items:center;gap:7px;
    font:700 calc(10px * var(--fs))/1 var(--mono);
    letter-spacing:.16em;text-transform:uppercase;
    padding:8px 11px;border-radius:0;
    background:transparent;color:var(--dph-ink-dark);
    border:1px solid rgba(21,23,23,.42);
  }
  .topbar .rt::before{
    content:"";width:6px;height:6px;background:var(--dph-signal);
    border:1px solid var(--dph-ink-dark);
    /* 常驻脉冲只动 opacity（合成器属性），并显式提层：
       不提层的话每帧都要顺手重绘整条顶栏。 */
    animation:pulse 1.9s ease-in-out infinite;
    will-change:opacity;
  }
  .tbtn{
    flex:0 0 auto;width:40px;height:40px;padding:0;margin:0;display:grid;place-items:center;
    border-radius:0;border:1px solid rgba(21,23,23,.28);
    background:transparent;color:var(--dph-ink-dark);
    box-shadow:none;
    transition:background .12s ease,color .12s ease,transform .12s ease;
  }
  .tbtn:active{background:var(--dph-ink-dark);color:var(--dph-paper);
    transform:none;box-shadow:none}
  .tbtn svg{width:19px;height:19px;display:block}
  .tbtn.more{margin-left:2px}
  @media (max-width:380px){.topbar .sub{display:none}}
  @media (max-width:330px){.topbar .rt{display:none}}

  /* ── 正文区 ──────────────────────────────────────────────────── */
  /* 底部只留一点呼吸：输入区是 flex 里的成员（不是浮层），
     给它 100px 的 padding 会在最后一条消息下面留一块空地。 */
  .list{padding:calc(var(--tbh) + 22px + var(--st,0px)) 16px 20px;gap:20px}

  /* 右缘竖排大字水印：参照侧栏的 wordmark，压到几乎看不见。
     消息行显式抬到它上面（z-index:1），否则水印会从卡片里透出来。 */
  .stage::before{
    content:"ENDFIELD";
    position:absolute;right:2px;top:46%;z-index:0;
    writing-mode:vertical-rl;text-orientation:mixed;
    font:800 calc(40px * var(--fs))/1 var(--sans);letter-spacing:.32em;
    color:rgba(238,243,239,.042);pointer-events:none;white-space:nowrap;
  }
  .row,.empty{position:relative;z-index:1}
  /* 行内自成一块：某一行在流式重绘时，重排/重绘不往整棵树外溢。
     paint 会把溢出裁掉 —— 行内不放浮层（放大图、长按菜单都挂在 body 上），
     所以裁不到东西，换来的是滚动和流式刷新时更小的脏区。 */
  .row{contain:layout paint}
  .stage::after{
    content:"";position:absolute;left:16px;right:16px;top:calc(var(--tbh) + 8px + var(--st,0px));
    height:1px;background:linear-gradient(90deg,var(--dph-rule),transparent 72%);
    pointer-events:none;z-index:0;
  }

  /* ── Markdown 正文 ──────────────────────────────────────────── */
  .body{
    font-family:var(--sans);
    font-size:calc(15px * var(--fs));line-height:1.78;
    color:var(--dph-ink);letter-spacing:.014em;
  }
  .body h1,.body h2,.body h3,.body h4{
    font-weight:800;letter-spacing:.04em;color:#fff;
  }
  .body h1{font-size:1.28em}
  .body h2{font-size:1.14em;padding-left:9px;border-left:3px solid var(--dph-signal);line-height:1.35}
  .body h3{font-size:1.03em;color:var(--dph-ink)}
  .body h4{font-size:.95em;color:var(--dph-muted)}
  .body strong{
    color:#fff;font-weight:800;
    background:linear-gradient(transparent 72%,rgba(242,245,0,.32) 0);
  }
  .body em{font-style:normal;color:var(--dph-signal);opacity:.92}
  .body s{opacity:.42}
  .body a{
    color:var(--dph-signal);text-decoration:none;font-weight:700;
    border-bottom:1px solid rgba(242,245,0,.35);
  }
  .body blockquote{
    border-left:3px solid var(--dph-signal);
    background:rgba(242,245,0,.05);
    border-radius:0;padding:8px 12px;margin-left:0;color:#cdd4d1;
  }
  .body code{
    color:#d9e2de;background:var(--dph-surface-strong);
    border:1px solid var(--dph-rule-soft);border-radius:0;
    padding:.1em .4em;font-family:var(--mono);font-size:.84em;
  }
  .body hr{border-top:1px solid var(--dph-rule)}
  .body th{background:var(--dph-surface);border-color:var(--dph-rule);color:#fff}
  .body td{border-color:var(--dph-rule-soft);color:var(--dph-ink)}
  .body li::marker{color:var(--dph-signal);font-weight:700}
  .body ul>li::marker{content:"▸ ";color:var(--dph-signal)}
  /* ── 文字标记（荧光笔 / 引用虚线）────────────────────────────── */
  /* 深色底上不能再用"半透明荧光笔 + 深字"，字会被糊住。改成低透明度
     色块 + 同色下划线，像用尺子在读数上划了一道记号。 */
  .mk{
    padding:.05em .1em;
    -webkit-box-decoration-break:clone;box-decoration-break:clone;
    box-shadow:inset 0 -2px 0 currentColor;
  }
  .mk.yellow{background:rgba(242,245,0,.18);color:#f2f500}
  .mk.green{background:rgba(114,214,181,.16);color:#72d6b5}
  .mk.blue{background:rgba(127,182,217,.16);color:#7fb6d9}
  .mk.pink{background:rgba(239,101,93,.16);color:#ef655d}
  .mk.gray{background:rgba(225,235,232,.14);color:#a7b0ad}
  .mk.q{background:none;box-shadow:none;border-bottom:1.5px dashed currentColor;padding:0}
  .mk.q.yellow{color:#d8db00}
  .mk.q.green{color:#72d6b5}
  .mk.q.blue{color:#8fc0e0}
  .mk.q.pink{color:#ef655d}
  .mk.q.gray{color:#89938f}

  .caretline{background:var(--dph-signal);width:7px;border-radius:0}
  .thinking i{background:var(--dph-signal);border-radius:0}

  /* ── 用户气泡：抬升层 + 切角，右对齐仍是"我说的" ──────────────── */
  .me{
    background:var(--dph-surface-strong);color:var(--dph-ink);
    border:1px solid var(--dph-rule);border-radius:0;
    padding:11px 15px;box-shadow:none;
    clip-path:polygon(12px 0,100% 0,100% calc(100% - 12px),calc(100% - 12px) 100%,0 100%,0 12px);
    font-size:calc(14.5px * var(--fs));
  }
  .meimg{border-radius:0;box-shadow:0 0 0 1px var(--dph-rule)}
  .mefile{
    background:var(--dph-surface);border-radius:0;
    box-shadow:inset 0 0 0 1px var(--dph-rule);
  }
  .mefile b{color:var(--dph-ink)}
  .mefile .ficon{color:var(--dph-muted)}

  /* ── AI 消息：等宽小标签 + 左细线 ─────────────────────────────── */
  .ai{
    border-left:1px solid var(--dph-rule);padding-left:14px;margin-left:0;
  }
  .ai.pending{border-left-color:var(--dph-signal-line)}
  .ai::before{
    content:"SYSTEM / 智能体";
    display:inline-flex;align-items:center;gap:6px;
    margin:0 0 10px;padding:0 0 5px 0;
    background:transparent;color:var(--dph-muted);
    font:700 calc(9.5px * var(--fs))/1 var(--mono);
    letter-spacing:.2em;text-transform:uppercase;border-radius:0;
    border-bottom:1px solid var(--dph-rule-soft);
    box-shadow:none;
  }
  .ai.pending::before{content:"SYSTEM / 执行中";color:var(--dph-signal)}

  /* ── 标签 / 折叠 ─────────────────────────────────────────────── */
  .meta{gap:5px;margin:0 0 9px}
  .tag{
    border-radius:0;background:transparent;
    border:1px solid var(--dph-rule);color:var(--dph-muted);
    font:700 calc(9.5px * var(--fs))/1 var(--mono);
    letter-spacing:.14em;text-transform:uppercase;padding:6px 8px;
  }
  .tag:active,.tag.active,.tag.open{
    background:var(--dph-signal);border-color:var(--dph-signal);color:var(--dph-ink-dark);
  }
  .tag.tool.failed{color:var(--dph-error);background:rgba(239,101,93,.10);border-color:rgba(239,101,93,.35)}
  .tag.toolsum.has-failed{color:var(--dph-error);background:rgba(239,101,93,.10)}
  .dot{border-radius:0;background:var(--dph-muted);width:6px;height:6px}
  .dot.running,.dot.success{background:var(--dph-signal)}
  .dot.awaitingConfirm{background:var(--dph-warn)}
  .dot.failed{background:var(--dph-error)}
  .dot.rejected{background:var(--dph-muted);opacity:.4}
  .quote{
    border-radius:0;border-left:2px solid var(--dph-signal);
    background:rgba(225,235,232,.05);color:var(--dph-muted);
  }
  .qbar{background:rgba(242,245,0,.06);border-left:3px solid var(--dph-signal);border-radius:0}
  .qbar .qt{color:var(--dph-ink);opacity:.9}
  .qbar .qx{background:var(--dph-surface-strong);color:var(--dph-ink);border-radius:0}

  /* ── 卡片 / 代码块：仪表盘读数条 ──────────────────────────────── */
  .fold,.codecard{
    border-radius:0;border:1px solid var(--dph-rule);
    background:var(--dph-surface);
  }
  .codecard .cap{
    padding:7px 8px 7px 11px;background:var(--dph-surface-strong);
    border-bottom:1px solid var(--dph-rule);border-radius:0;
  }
  .codecard .lang{
    font-family:var(--mono);letter-spacing:.18em;text-transform:uppercase;
    color:var(--dph-signal);font-weight:700;
  }
  .codecard pre{background:#0c1011}
  .codecard pre code{color:var(--dph-ink);font-family:var(--mono)}
  .codecard .pv{border-top:1px solid var(--dph-rule);background:var(--dph-field)}
  .codecard .pv .pbar{
    background:var(--dph-surface);border-bottom:1px solid var(--dph-rule);
    color:var(--dph-muted);font-family:var(--mono);letter-spacing:.16em;
  }
  .mini{
    border-radius:0;color:var(--dph-muted);
    font:700 calc(9.5px * var(--fs))/1 var(--mono);
    letter-spacing:.14em;text-transform:uppercase;padding:6px 8px;
    border:1px solid transparent;
  }
  .mini:active,.mini.done{background:rgba(242,245,0,.10);color:var(--dph-signal);border-color:var(--dph-signal-line)}
  .codecard .jsout{background:#0c1011;border-top:1px solid var(--dph-rule);color:var(--dph-ink)}
  .codecard .cnote{border-top:1px dashed var(--dph-rule);color:var(--dph-muted)}
  .codecard .jsout .ln.err{color:var(--dph-error)}
  .fold{border-left:3px solid var(--dph-signal)}
  .fold .fttl{
    font:700 calc(9.5px * var(--fs))/1 var(--mono);
    letter-spacing:.18em;color:var(--dph-muted);
    border-bottom-color:var(--dph-rule);
  }
  .fold pre{color:#b9c2be;font-family:var(--mono)}
  .fold pre + pre{border-top:1px dashed var(--dph-rule)}
  .reasoning pre{font-family:var(--sans)}
  .mdimg img,.codecard .imgc img{border-radius:0;border:1px solid var(--dph-rule);background:var(--dph-field)}
  .mdimg .mdcap{font-family:var(--mono);letter-spacing:.1em;text-transform:uppercase;font-size:calc(10px * var(--fs))}
  .lchip,.fchip{
    border-radius:0;background:var(--dph-surface-strong);
    border:1px solid var(--dph-rule);color:var(--dph-ink);
    font-family:var(--mono);font-size:calc(11px * var(--fs));
  }
  .lchip{color:var(--dph-signal)}
  .nofs,.nofs2{border-radius:0;background:var(--dph-surface);border:1px solid var(--dph-rule)}
  .err{
    border-radius:0;background:rgba(239,101,93,.10);color:var(--dph-error);
    border:1px solid rgba(239,101,93,.35);
  }
  .mn-rect{fill:var(--dph-surface-strong);stroke:var(--dph-rule)}
  .mn-root{fill:var(--dph-signal)}
  .mn-t{fill:var(--dph-ink)}
  .mn-t-root{fill:var(--dph-ink-dark)}
  .mn-link{stroke:var(--dph-rule)}

  /* ── 浮动控件 ────────────────────────────────────────────────── */
  .jump{
    border-radius:0;background:var(--dph-surface);border:1px solid var(--dph-rule);
    color:var(--dph-ink);box-shadow:0 6px 18px rgba(0,0,0,.45);
    clip-path:var(--dph-cut-sm);
  }
  .jump.on:hover{background:var(--dph-surface-strong)}
  .jump .badge{
    border:0;background:var(--dph-signal);border-radius:0;
    transform:rotate(45deg);
  }
  .jump.dirty .badge{transform:rotate(45deg) scale(1);animation:pulse 1.4s ease-in-out infinite}
  .jhint{
    border-radius:0;background:var(--dph-signal);color:var(--dph-ink-dark);
    font:700 calc(9.5px * var(--fs))/1 var(--mono);letter-spacing:.16em;text-transform:uppercase;
  }
  .scard,.cmenu,.mention{
    border-radius:0;background:var(--dph-surface);border:1px solid var(--dph-rule);
    box-shadow:0 16px 34px rgba(0,0,0,.55);
  }
  .scard{font-family:var(--mono);font-size:calc(11px * var(--fs));letter-spacing:.04em;color:var(--dph-ink)}
  .cmenu button{
    border-radius:0;color:var(--dph-ink);
    font:700 calc(11px * var(--fs))/1 var(--mono);
    letter-spacing:.1em;text-transform:uppercase;
  }
  .cmenu button:active,.cmenu button:hover{background:var(--dph-signal);color:var(--dph-ink-dark)}
  .cmenu button.danger{color:var(--dph-error)}
  .cmenu button.danger:active{background:rgba(239,101,93,.14);color:var(--dph-error)}
  .mitem{border-radius:0}
  .mitem .ic{
    background:rgba(242,245,0,.12);color:var(--dph-signal);border-radius:0;
    font:700 calc(9.5px * var(--fs))/1 var(--mono);letter-spacing:.1em;
    border:1px solid var(--dph-signal-line);
  }
  .mitem .tt{font:700 calc(12.5px * var(--fs))/1.35 var(--sans);color:var(--dph-ink)}
  .mitem .nt{font:calc(10.5px * var(--fs))/1.4 var(--mono);color:var(--dph-muted)}
  .mitem.cur,.mitem:active{background:rgba(242,245,0,.10)}
  .mempty{font-family:var(--mono);letter-spacing:.1em;text-transform:uppercase;font-size:calc(10.5px * var(--fs))}

  /* 滑动定位器：细导轨 + 信号滑块 */
  .srail{border-radius:0;background:rgba(225,235,232,.07);box-shadow:inset 0 0 0 1px var(--dph-rule-soft)}
  .sthumb{border-radius:0;background:var(--dph-muted)}
  .stick{border-radius:0;background:var(--dph-muted);opacity:.3}
  .scrub.dragging .sthumb,.stick.cur{background:var(--dph-signal)}
  .scrub.dragging .sthumb{opacity:1}

  /* 到顶/到底的柔光：换成信号黄，别再用主色泛白 */
  .edgeglow.top{background:linear-gradient(to bottom,rgba(242,245,0,.16),transparent)}
  .edgeglow.bottom{background:linear-gradient(to top,rgba(242,245,0,.14),transparent)}

  /* ── 空态（首页）：等宽 kicker + 大标题 + 预览章 ───────────────── */
  .empty{
    margin:auto;text-align:center;padding:22px 10px;max-width:360px;
    display:flex;flex-direction:column;align-items:center;
  }
  .empty .kicker{
    display:flex;align-items:center;gap:8px;margin:0 0 22px;
    color:var(--dph-muted);
    font:700 calc(9.5px * var(--fs))/1 var(--mono);
    letter-spacing:.22em;text-transform:uppercase;white-space:nowrap;
  }
  .empty .kicker i{width:5px;height:5px;background:var(--dph-signal);flex:0 0 auto}
  .empty .kicker::after{
    content:"";display:block;width:104px;height:1px;
    background:linear-gradient(90deg,var(--dph-rule-strong),transparent);
  }
  .empty .tline{display:flex;align-items:center;justify-content:center;gap:10px;flex-wrap:wrap}
  .empty .glyph{
    width:30px;height:30px;margin:0;padding:0;border:1px solid var(--dph-signal);
    background:rgba(242,245,0,.10);border-radius:0;
    display:grid;place-items:center;color:var(--dph-signal);
    font:800 calc(13px * var(--fs))/1 var(--mono);
    transform:none;flex:0 0 auto;
  }
  .empty .glyph svg{width:16px;height:16px;display:block}
  .empty h1{
    margin:0;font:800 calc(25px * var(--fs))/1.28 var(--sans);
    letter-spacing:.06em;color:#fff;text-shadow:none;
  }
  .empty .badge{
    display:inline-block;padding:3px 7px;border-radius:0;
    background:var(--dph-surface-strong);border:1px solid var(--dph-rule);
    color:var(--dph-ink);
    font:700 calc(9px * var(--fs))/1 var(--mono);
    letter-spacing:.18em;text-transform:uppercase;align-self:center;
  }
  .empty p{
    margin:14px 0 0;max-width:300px;
    color:var(--dph-muted);font:400 calc(12.5px * var(--fs))/1.75 var(--sans);
    letter-spacing:.02em;
  }

  /* ── 输入区：一块直角切边的控制台面板 ────────────────────────── */
  .composer{padding:0 14px calc(16px + env(safe-area-inset-bottom))}
  .box{
    position:relative;
    border-radius:0;background:var(--dph-surface);
    border:1px solid var(--dph-rule);
    padding:8px 8px 7px;gap:3px;
    box-shadow:0 10px 26px rgba(0,0,0,.42);
    clip-path:var(--dph-cut);
    -webkit-backdrop-filter:none;backdrop-filter:none;
    transition:border-color .18s ease,background .18s ease;
  }
  /* 左上角信号标：两根短线拼成的 HUD 角标（横 + 竖），
     比原来一条 46px 的实心短线更像仪表盘刻度，也不会跟附件芯片粘成一条。 */
  .box::before{
    content:"";position:absolute;left:7px;top:7px;width:13px;height:2px;
    background:var(--dph-signal);pointer-events:none;
  }
  .box::after{
    content:"";position:absolute;left:7px;top:7px;width:2px;height:13px;
    background:var(--dph-signal);pointer-events:none;
  }
  .box:focus-within{
    border-color:var(--dph-signal-line);
    box-shadow:0 10px 26px rgba(0,0,0,.42),0 0 0 1px rgba(242,245,0,.10);
  }
  body.scrolling .box,body.kb .box{background:var(--dph-surface)}
  textarea#i{
    height:64px;min-height:64px;padding:9px 10px 2px;
    color:var(--dph-ink);caret-color:var(--dph-signal);
    font:calc(15px * var(--fs))/22px var(--sans);letter-spacing:.01em;
  }
  textarea#i::placeholder{color:#65706d;opacity:1}
  .toolrow{gap:8px;padding:0 2px 1px}
  .icon{
    border-radius:0;width:32px;height:32px;color:var(--dph-muted);
    border:1px solid transparent;
  }
  .icon:active{background:rgba(242,245,0,.10);border-color:var(--dph-signal-line);color:var(--dph-signal)}
  #bFile{border-color:var(--dph-rule-soft)}
  .toolrow .model{
    margin-left:auto;margin-right:2px;max-width:52%;
    padding:6px 9px;border-radius:0;
    background:transparent;border:1px solid var(--dph-rule);
    color:var(--dph-muted);
    font:700 calc(9.5px * var(--fs))/1.3 var(--mono);
    letter-spacing:.14em;text-transform:uppercase;
  }
  .toolrow .model:active{background:rgba(242,245,0,.08);border-color:var(--dph-signal-line);color:var(--dph-signal)}
  .send{
    width:38px;height:32px;border-radius:0;
    background:var(--dph-signal);color:var(--dph-ink-dark);
    border:1px solid var(--dph-signal);
    clip-path:polygon(0 0,calc(100% - 9px) 0,100% 9px,100% 100%,0 100%);
    box-shadow:none;
    transition:opacity .16s ease,background .16s ease;
  }
  .send:active:not(:disabled){transform:none;background:#fbff52}
  /* 空输入时不要用斜纹：斜纹是"禁用/危险"的信号，一屏只该有一处。
     这里压成一块暗面板 + 暗箭头，安静地"没电"。 */
  .send:disabled{
    opacity:1;
    background:#1a2021;
    border-color:var(--dph-rule);color:#6f7a76;
  }
  .send.stop{background:var(--dph-error);border-color:var(--dph-error);color:#fff}
  .attachbar{color:var(--dph-muted);font-family:var(--mono);letter-spacing:.1em}
  .att{
    background:var(--dph-surface);border-radius:0;
    box-shadow:inset 0 0 0 1px var(--dph-rule);
  }
  .att b{color:var(--dph-ink)}
  .att img{border-radius:0}
  .att .attx{background:var(--dph-surface-strong);color:var(--dph-ink);border-radius:0}
  .attachbar .hint{color:var(--dph-muted);font-size:calc(9.5px * var(--fs));letter-spacing:.14em;text-transform:uppercase}

  /* 文件预览 / 图片放大：同一套铬 */
  .fview .fhead button{border-radius:0;background:transparent;border:1px solid var(--dph-rule);color:var(--dph-ink);
    font-family:var(--mono);letter-spacing:.12em;text-transform:uppercase;font-size:calc(10px * var(--fs))}
  .fview .fhead button:active{background:var(--dph-signal);border-color:var(--dph-signal);color:var(--dph-ink-dark)}
  .fview .fname{letter-spacing:.02em}
  .fview .fmeta{font-family:var(--mono);letter-spacing:.1em;font-size:calc(9.5px * var(--fs))}
  .zoom .cap{font-family:var(--mono);letter-spacing:.1em;color:var(--dph-muted)}

  /* ── 深色语义兜底 ──────────────────────────────────────────────
     主题本身是深色（onTheme 现在报 dark:true，沙箱 iframe 才会用深底），
     基础样式里 html[data-dark="1"] 的老规则有些会盖掉这里的取值，逐条纠回来。 */
  html[data-dark="1"] .body a{border-bottom-color:rgba(242,245,0,.35)}
  html[data-dark="1"] .err{background:rgba(239,101,93,.10);color:var(--dph-error)}
  html[data-dark="1"] .tag.tool.failed,html[data-dark="1"] .tag.toolsum.has-failed{
    background:rgba(239,101,93,.10);color:var(--dph-error);
  }
  html[data-dark="1"] .codecard .pv{border-top-color:var(--dph-rule);background:var(--dph-field)}
  html[data-dark="1"] .codecard .pv .pbar{
    color:var(--dph-muted);background:var(--dph-surface);border-bottom-color:var(--dph-rule);
  }
  html[data-dark="1"] .jump{box-shadow:0 6px 18px rgba(0,0,0,.5)}

  /* toast / 提示条走同一层 */
  .dph-toast{
    background:var(--dph-paper)!important;color:var(--dph-ink-dark)!important;
    border-radius:0!important;
    border-left:4px solid var(--dph-signal);
    font:700 calc(11.5px * var(--fs))/1.5 var(--mono)!important;
    letter-spacing:.06em;
  }

  /* ── 安全区 ────────────────────────────────────────────────────── */
  :root{--st:env(safe-area-inset-top,0px)}
  .tbtn{width:40px;height:40px;margin:0}

  /* 宽屏（平板/桌面预览）：正文收成一条读数栏，别让长行横跨整块屏 */
  @media (min-width:720px){
    .list{padding-left:28px;padding-right:28px}
    .row{width:100%;max-width:860px;margin-left:auto;margin-right:auto}
    .empty{max-width:420px}
  }

  /* ── 动效：参照的入场语言 ─────────────────────────────────────── */
  @keyframes dph-header-enter{
    from{opacity:0;transform:translateY(-20px)}
    to{opacity:1;transform:translateY(0)}
  }
  @keyframes dph-index-load{
    from{transform:scaleX(0);opacity:0}
    to{transform:scaleX(1);opacity:1}
  }
  /* 入场只用 opacity + transform：这两个属性走合成器，不触发重排重绘。
     之前用 clip-path 做擦入，等于每帧给整行重新算一次裁切路径 ——
     流式输出时一行行往下刷，那是白送给 GPU 的开销。 */
  @keyframes dph-record-enter{
    from{opacity:0;transform:translateY(14px)}
    to{opacity:1;transform:none}
  }
  @media (prefers-reduced-motion:no-preference){
    /* 不带 fill：动画放完就让浏览器收掉这一行的合成层，
       长会话里几十行各挂一个层，内存会白白涨一大截。 */
    .row{animation:dph-record-enter .4s cubic-bezier(.2,.8,.2,1)!important}
    .empty{animation:dph-record-enter .46s cubic-bezier(.2,.8,.2,1)}
  }
</style>
</head>
<body>

<div class="frame" aria-hidden="true"></div>


<div class="topbar"><button class="tbtn" type="button" data-nav="back" aria-label="返回"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round"><path d="M15 5l-7 7 7 7"/></svg></button><span class="sq"></span><b>终末地</b><span class="sub">ENDFIELD<br>HARNESS</span><span class="rt">在线</span><button class="tbtn more" type="button" data-nav="more" aria-label="更多"><svg viewBox="0 0 24 24" fill="currentColor"><circle cx="5.5" cy="12" r="2"/><circle cx="12" cy="12" r="2"/><circle cx="18.5" cy="12" r="2"/></svg></button><script>
(function () {
  var tEl = null, tTimer = 0;
  function tip(msg) {
    try {
      if (!tEl) {
        tEl = document.createElement("div");
        tEl.style.cssText = "position:fixed;left:16px;right:16px;top:calc(66px + var(--st,0px));z-index:99;" +
          "padding:10px 14px;background:var(--dph-paper,#f2f3ef);color:var(--dph-ink-dark,#151717);" +
          "border-left:4px solid var(--dph-signal,#f2f500);border-radius:0;text-align:left;" +
          "font:700 11.5px/1.55 var(--mono,monospace);letter-spacing:.04em;" +
          "box-shadow:0 10px 26px rgba(0,0,0,.5);opacity:0;transition:opacity .2s ease;" +
          "pointer-events:none;word-break:break-all";
        document.body.appendChild(tEl);
      }
      tEl.textContent = msg;
      requestAnimationFrame(function () { tEl.style.opacity = "1"; });
      clearTimeout(tTimer);
      tTimer = setTimeout(function () { tEl.style.opacity = "0"; }, 4200);
    } catch (e) {}
  }
  function hasA(n) {
    return !!(window.App && typeof window.App[n] === "function");
  }
  function nav(kind) {
    /* 文档给的就是 App.back() / App.more()：优先直呼，参数一个都不带。 */
    if (hasA(kind)) {
      try { window.App[kind](); return; } catch (e) { tip("App." + kind + "() 报错：" + e.message); return; }
    }
    var alias = { back: ["goBack", "pop"], more: ["menu", "openMenu"] }[kind] || [];
    for (var i = 0; i < alias.length; i++) {
      if (hasA(alias[i])) { try { window.App[alias[i]](); return; } catch (e) {} }
    }
    /* App 上没有这个动作：先把诊断亮出来，再试一次桥（桥通常也没有 back） */
    tip("点了没反应时看这里 → App." + kind + " 不存在。当前 App 只有 [" +
        (window.App ? Object.keys(window.App).join(", ") : "空") + "]");
    var br = window.flutter_inappwebview;
    if (br && typeof br.callHandler === "function") {
      try { br.callHandler(kind); } catch (e) {}
    }
  }
  window.__nav = nav;

  /* 诊断小圆点：App.back 缺失时按钮右上角亮红点，一眼能看出是接口没到 */
  function dot(btn) {
    var d = document.createElement("i");
    d.style.cssText = "position:absolute;top:-2px;right:-2px;width:8px;height:8px;border-radius:50%;" +
      "background:#ef655d;box-shadow:0 0 0 2px #f2f3ef;pointer-events:none";
    btn.style.position = "relative";
    btn.appendChild(d);
  }
  setTimeout(function () {
    if (!hasA("back")) {
      var b = document.querySelector('.tbtn[data-nav="back"]');
      if (b) dot(b);
    }
  }, 1800);

  /* 长按标题 = 打印诊断 */
  (function () {
    var t = document.querySelector(".topbar b");
    if (!t) return;
    var timer = 0;
    t.addEventListener("pointerdown", function () {
      timer = setTimeout(function () {
        tip("App 键：" + (window.App ? Object.keys(window.App).join(", ") : "空") +
            " ｜ back=" + (hasA("back") ? "有" : "无") + " more=" + (hasA("more") ? "有" : "无"));
      }, 650);
    });
    ["pointerup", "pointercancel", "pointerleave"].forEach(function (k) {
      t.addEventListener(k, function () { clearTimeout(timer); });
    });
  })();

  /* 双通道绑定：click 之外再加 pointerup，某些 WebView 里 click 会被吞 */
  (function () {
    var last = 0;
    function fire(kind) {
      var now = Date.now();
      if (now - last < 400) return;
      last = now;
      nav(kind);
    }
    var bs = document.querySelectorAll(".tbtn[data-nav]");
    for (var i = 0; i < bs.length; i++) {
      (function (b) {
        var kind = b.getAttribute("data-nav");
        b.addEventListener("click", function (e) { e.preventDefault(); fire(kind); });
        b.addEventListener("pointerup", function () { fire(kind); });
        b.addEventListener("touchend", function () { fire(kind); }, { passive: true });
      })(bs[i]);
    }
  })();
})();
</script></div>
<div id="boot" aria-hidden="true">
  <div class="mtx"></div>
  <div class="glow"></div>
  <div class="scan"></div>
  <div class="strip"></div>
  <div class="br tl"></div><div class="br tr"></div><div class="br bl"></div><div class="br br2"></div>
  <div class="core">
    <div class="kick"><i></i><span>DPH / HARNESS TERMINAL</span></div>
    <div class="mark">
      <div class="zh">终末地</div>
      <div class="en">ENDFIELD</div>
    </div>
    <div class="rule"><i id="bootRule"></i></div>
    <div class="sub">ENDFIELD × DEEPSEEK HARNESS · WEB TERMINAL</div>
  </div>
  <div class="log" id="bootLog"></div>
  <div class="ftr">
    <div class="segs" id="bootSegs"></div>
    <div class="pct"><span id="bootPct">00</span><b>%</b></div>
  </div>
  <div class="hazard"></div>
</div>
<style>
  /* ── 启动动画：终末地终端开机序列 ──────────────────────────────
     构造顺序：黑场 → 矩阵漂移 → 扫描线 → 四角标 → 字号牌揭幕 →
     等宽日志逐行落地 → 分段进度 → SYSTEM READY 闪一帧 → 向上擦除退场。 */
  #boot{
    position:fixed;inset:0;z-index:300;overflow:hidden;
    background:var(--dph-field,#0a0d0e);
    color:var(--dph-ink,#eef3ef);
    font-family:var(--sans,"Segoe UI",sans-serif);
    /* 退场用整块上移，不用 clip-path：裁切动画在部分 WebView 的合成层上
       会把不透明底的绘制拆开，出现"底透了、子元素还在"的穿帮。 */
    transform:translateY(0);
    transition:transform .62s cubic-bezier(.72,0,.2,1);
    will-change:transform;
  }
  #boot.out{transform:translateY(-100%)}
  #boot .mtx{
    position:absolute;inset:-12%;pointer-events:none;
    background:
      repeating-linear-gradient(90deg,rgba(225,235,232,.05) 0 1px,transparent 1px 72px),
      repeating-linear-gradient(0deg,rgba(225,235,232,.04) 0 1px,transparent 1px 72px),
      linear-gradient(90deg,transparent 0 19%,rgba(255,255,255,.03) 19% 32%,transparent 32% 61%,rgba(242,245,0,.07) 61% 69%,transparent 69%),
      linear-gradient(0deg,transparent 0 16%,rgba(255,255,255,.025) 16% 29%,transparent 29% 58%,rgba(255,255,255,.016) 58% 78%,transparent 78%);
    transform:rotate(-7deg);
    animation:dph-boot-mtx 9s ease-in-out infinite alternate;
    will-change:transform;
  }
  #boot .glow{
    position:absolute;inset:0;pointer-events:none;
    background:radial-gradient(circle at 78% 18%,rgba(242,245,0,.10),transparent 46%),
               radial-gradient(circle at 12% 92%,rgba(114,214,181,.05),transparent 40%);
  }
  /* 扫描线：一条实线 + 下方 64px 的淡光带，看起来才是"扫过去" */
  #boot .scan{
    position:absolute;left:0;right:0;top:0;height:2px;
    background:linear-gradient(90deg,transparent,rgba(242,245,0,.9) 14%,var(--dph-signal,#f2f500) 50%,rgba(242,245,0,.9) 86%,transparent);
    box-shadow:0 0 26px 2px rgba(242,245,0,.55);
    animation:dph-boot-scan 2.6s cubic-bezier(.55,0,.45,1) infinite;
    will-change:transform;
  }
  #boot .scan::after{
    content:"";position:absolute;left:0;right:0;top:2px;height:64px;
    background:linear-gradient(to bottom,rgba(242,245,0,.13),transparent);
  }
  #boot .strip{
    position:absolute;left:0;right:0;top:0;height:4px;
    background:linear-gradient(90deg,#151717 0 18%,var(--dph-signal,#f2f500) 18% 23%,#151717 23% 100%);
  }
  #boot .br{position:absolute;width:26px;height:26px;border-color:var(--dph-signal,#f2f500);border-style:solid;border-width:0}
  #boot .tl{left:18px;top:18px;border-left-width:2px;border-top-width:2px}
  #boot .tr{right:18px;top:18px;border-right-width:2px;border-top-width:2px}
  #boot .bl{left:18px;bottom:18px;border-left-width:2px;border-bottom-width:2px}
  #boot .br2{right:18px;bottom:18px;border-right-width:2px;border-bottom-width:2px}

  #boot .core{position:absolute;left:34px;right:34px;top:50%;transform:translateY(-56%)}
  #boot .kick{
    display:flex;align-items:center;gap:8px;margin:0 0 20px;
    color:var(--dph-muted,#89938f);
    font:700 9.5px/1 var(--mono,monospace);
    letter-spacing:.26em;text-transform:uppercase;
    opacity:0;animation:dph-boot-in .5s ease .1s forwards;
  }
  #boot .kick i{width:5px;height:5px;background:var(--dph-signal,#f2f500);display:block;flex:0 0 auto}
  #boot .kick::after{content:"";width:88px;height:1px;background:linear-gradient(90deg,rgba(225,235,232,.4),transparent);display:block}

  #boot .mark .zh{
    font:800 40px/1 var(--sans,"Segoe UI",sans-serif);
    letter-spacing:.16em;color:#fff;
    clip-path:inset(0 0 100% 0);
    animation:dph-boot-reveal .62s cubic-bezier(.2,.8,.2,1) .18s forwards;
  }
  #boot .mark .en{
    margin-top:6px;
    font:800 62px/.92 var(--sans,"Segoe UI",sans-serif);
    letter-spacing:.06em;
    color:transparent;-webkit-text-stroke:1px rgba(242,245,0,.9);
    clip-path:inset(0 0 100% 0);
    animation:dph-boot-reveal .72s cubic-bezier(.2,.8,.2,1) .3s forwards;
  }
  #boot .rule{margin:20px 0 10px;height:2px;background:rgba(225,235,232,.12);overflow:hidden}
  #boot .rule i{display:block;height:100%;width:0;background:var(--dph-signal,#f2f500);
    transform-origin:left;transition:width 1.9s cubic-bezier(.5,0,.2,1)}
  #boot .sub{
    color:var(--dph-muted,#89938f);
    font:700 8.5px/1.4 var(--mono,monospace);
    letter-spacing:.22em;text-transform:uppercase;
    opacity:0;animation:dph-boot-in .6s ease .62s forwards;
  }

  #boot .log{
    position:absolute;left:34px;right:34px;bottom:96px;
    font:700 9.5px/1.9 var(--mono,monospace);
    letter-spacing:.14em;text-transform:uppercase;color:var(--dph-muted,#89938f);
  }
  #boot .log div{
    display:flex;gap:8px;opacity:0;transform:translateX(-6px);
    animation:dph-boot-line .3s ease forwards;
  }
  #boot .log b{color:var(--dph-signal,#f2f500);font-weight:700}
  #boot .log u{text-decoration:none;opacity:.45}
  #boot .log .ok{color:#b9c2be}
  #boot .log .cur{color:#fff}
  #boot .log .cur::after{
    content:"\2588";color:var(--dph-signal,#f2f500);margin-left:2px;
    animation:blink 1s steps(2,start) infinite;
  }

  #boot .ftr{position:absolute;left:34px;right:34px;bottom:52px;display:flex;align-items:center;gap:12px}
  #boot .segs{display:flex;gap:3px;flex:1 1 auto}
  #boot .segs i{flex:1 1 0;height:5px;background:rgba(225,235,232,.12);transition:background .18s ease}
  #boot .segs i.on{background:var(--dph-signal,#f2f500)}
  #boot .pct{
    flex:0 0 auto;display:flex;align-items:baseline;gap:3px;
    color:#fff;font:800 15px/1 var(--mono,monospace);letter-spacing:.06em;
  }
  #boot .pct b{font-size:9px;color:var(--dph-muted,#89938f);letter-spacing:.1em}
  #boot.done .pct span{color:var(--dph-signal,#f2f500)}
  #boot.done .hazard{opacity:1}
  #boot .hazard{
    position:absolute;left:0;right:0;bottom:0;height:7px;
    background:repeating-linear-gradient(135deg,#151717 0 5px,var(--dph-signal,#f2f500) 5px 10px);
    opacity:.85;
  }

  @keyframes dph-boot-in{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:none}}
  @keyframes dph-boot-reveal{from{clip-path:inset(0 0 100% 0)}to{clip-path:inset(0 0 0 0)}}
  @keyframes dph-boot-line{to{opacity:1;transform:none}}
  @keyframes dph-boot-mtx{from{transform:rotate(-7deg) translate(0,-10px)}to{transform:rotate(-5deg) translate(-22px,14px)}}
  @keyframes dph-boot-scan{from{transform:translateY(-4px)}to{transform:translateY(100vh)}}
  @media (max-width:360px){
    #boot .core{left:24px;right:24px}
    #boot .mark .zh{font-size:33px}
    #boot .mark .en{font-size:48px}
    #boot .log,#boot .ftr{left:24px;right:24px}
    #boot .log{bottom:88px}
  }
  @media (prefers-reduced-motion:reduce){
    #boot .scan{animation:none;opacity:.3}
    #boot .mtx{animation:none}
  }
</style>
<script>
(function () {
  var el = document.getElementById("boot");
  if (!el) return;
  /* 只有第一次进这个主题才放一遍开机动画；?boot 强制重播，方便调参 */
  var KEY = "dph.boot.v1", seen = false;
  try { seen = localStorage.getItem(KEY) === "1"; } catch (e) {}
  if (seen && location.search.indexOf("boot") < 0) {
    el.remove();
    document.documentElement.setAttribute("data-booted", "1");
    return;
  }

  var reduce = !!(window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches);

  var segsEl = document.getElementById("bootSegs");
  var pctEl = document.getElementById("bootPct");
  var ruleEl = document.getElementById("bootRule");
  var logEl = document.getElementById("bootLog");
  var N = 12, filled = 0, segs = [];
  for (var i = 0; i < N; i++) {
    var s = document.createElement("i");
    segsEl.appendChild(s);
    segs.push(s);
  }

  /* 开机日志：等宽、全大写，右列是状态码 */
  var lines = [
    ["BOOT SEQUENCE", "INIT", ""],
    ["FIELD RENDER", "OK", ""],
    ["SIGNAL LINK", "ACTIVE", ""],
    ["OPERATOR AUTH", "GUEST", ""],
    ["TERMINAL READY", "OK", "cur"]
  ];
  var li = 0;

  function pushLog() {
    if (li >= lines.length) return;
    var row = document.createElement("div");
    row.innerHTML = "<b>&gt;</b><span>" + lines[li][0] + "</span>" +
      "<u>" + new Array(Math.max(2, 22 - lines[li][0].length)).join(".") + "</u>" +
      "<span class='" + (lines[li][1] === "OK" ? "ok" : "") + "'>" + lines[li][1] + "</span>";
    if (lines[li][2]) row.className = lines[li][2];
    logEl.appendChild(row);
    /* 光标永远停在最新一行 */
    if (li > 0) {
      var prev = logEl.children[li - 1];
      if (prev && prev.className === "cur") prev.className = "";
    }
    li++;
    setTimeout(pushLog, reduce ? 40 : 220);
  }

  function tick() {
    if (filled >= N) return;
    segs[filled].className = "on";
    filled++;
    pctEl.textContent = filled === N ? "100" : ("0" + Math.floor(filled / N * 100)).slice(-2);
    if (filled >= N) el.classList.add("done");
    if (filled < N) setTimeout(tick, reduce ? 16 : 140);
  }

  /* 节奏：0.24s 起进度与日志 → 约 1.9s 满格 → 停一拍再擦除退场 */
  setTimeout(function () {
    if (ruleEl) ruleEl.style.width = "100%";
    pushLog();
    tick();
  }, reduce ? 0 : 240);

  var total = reduce ? 420 : 2320;
  setTimeout(function () {
    el.classList.add("out");
    try { localStorage.setItem(KEY, "1"); } catch (e) {}
    /* 顶栏的入场动效等这个标记：擦除上来正好接住它 */
    document.documentElement.setAttribute("data-booted", "1");
    setTimeout(function () {
      el.remove();
    }, 700);
  }, total);
})();
</script>

<div class="stage">
  <div class="list" id="list">
    <div class="empty" id="empty">
      <div class="kicker"><i></i><span id="ekick">ENDFIELD / HARNESS THEME</span></div>
      <div class="tline">
        <div class="glyph"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="square"><path d="M5 3.5l7 8.5-7 8.5"/><path d="M13 20.5h6"/></svg></div>
        <h1 id="greet">探索未至之境</h1>
        <span class="badge">预览版</span>
      </div>
      <p>向智能体发送中文问候，或描述你要构建的内容</p>
    </div>
  </div>

  <div class="scrub" id="scrub">
    <div class="srail" id="srail">
      <div class="sticks" id="sticks"></div>
      <div class="sthumb" id="sthumb"></div>
    </div>
    <div class="scard" id="scard"><span id="scardTxt"></span></div>
  </div>
</div>

<div class="composer">
  <div class="mention" id="mention" hidden></div>

  <button class="jump" id="jump" aria-label="回到底部（可拖拽到顶/到底）">
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
      <path d="M12 5v14M6 13l6 6 6-6"/>
    </svg>
    <span class="badge"></span>
    <span class="jhint" id="jhint"></span>
  </button>

  <div class="qbar" id="qbar">
    <div class="qt" id="qbarTxt"></div>
    <button class="qx" id="qbarX" aria-label="删除引用">×</button>
  </div>

  <div class="attachbar" id="attachbar">
    <div class="atts" id="atts"></div>
  </div>

  <div class="box" id="box">
    <textarea id="i" rows="1" placeholder="描述你想要构建的内容" enterkeyhint="send"></textarea>
    <div class="toolrow">
      <button class="icon" id="bFile" aria-label="添加文件">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" stroke-linecap="round">
          <path d="M12 5v14M5 12h14"/>
        </svg>
      </button>
      <button class="icon" id="bImg" aria-label="添加图片">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round">
          <rect x="3" y="4.5" width="18" height="15" rx="2.6"/>
          <circle cx="8.6" cy="9.8" r="1.5"/>
          <path d="M4 16.5l4.6-4.2a1.7 1.7 0 012.3 0L16 17"/>
        </svg>
      </button>
      <span class="model" id="model" role="button">—</span>
      <button class="send" id="s" aria-label="发送">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <path d="M12 19V5M6 11l6-6 6 6"/>
        </svg>
      </button>
    </div>
  </div>
</div>

<script>
(function () {
  "use strict";

  /* ═══ 0. 环境 ═══════════════════════════════════════════════════ */
  var HAS_BRIDGE = !!(window.flutter_inappwebview &&
                      typeof window.flutter_inappwebview.callHandler === "function");

  var $ = function (id) { return document.getElementById(id); };
  var listEl = $("list"), emptyEl = $("empty"), modelEl = $("model"),
      inputEl = $("i"), sendEl = $("s"), boxEl = $("box"),
      jumpEl = $("jump"), attachBar = $("attachbar"), attsEl = $("atts"),
      qbarEl = $("qbar"), qbarTxt = $("qbarTxt"), qbarX = $("qbarX"),
      jhintEl = $("jhint"),
      stageEl = document.querySelector(".stage");

  var ICON_SEND = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 19V5M6 11l6-6 6 6"/></svg>';
  var ICON_STOP = '<svg viewBox="0 0 24 24" fill="currentColor"><rect x="6" y="6" width="12" height="12" rx="2.4"/></svg>';

  var raf = window.requestAnimationFrame || function (fn) { return setTimeout(fn, 16); };

  /* ── 全局错误兜底：页面脚本出意外时弹一句 toast，绝不让 WebView 白屏 ── */
  var lastErrToast = 0;
  window.addEventListener("error", function () {
    var now = Date.now();
    if (now - lastErrToast > 5000) {
      lastErrToast = now;
      try { toast("页面出了点小状况，已自动恢复"); } catch (e) {}
    }
  });
  window.addEventListener("unhandledrejection", function () {});

  /* 原生弹窗一律禁掉：WebView 里的 alert/confirm/prompt 会拉起 Flutter
     一侧的对话框，看着像"应用报错弹窗"，还会把正在滚动的列表冻住。
     统一折成页面内 toast；confirm 保守返回 false，prompt 返回 null。 */
  (function () {
    var say = function (t) { try { toast(String(t == null ? "" : t)); } catch (e) {} };
    window.alert = function (m) { say(m); };
    window.confirm = function (m) { say((m || "") + "（已自动取消）"); return false; };
    window.prompt = function (m) { say((m || "") + "（页面用输入框代替弹窗）"); return null; };
  })();

  /* ── 禁掉双指缩放：固定视图被放大后整页排版就散了 ────────────── */
  (function () {
    var stop = function (e) { if (e.cancelable) e.preventDefault(); };
    document.addEventListener("gesturestart", stop, { passive: false });
    document.addEventListener("gesturechange", stop, { passive: false });
    document.addEventListener("gestureend", stop, { passive: false });
    /* 双指按下/移动：拦住手势缩放（单指滚动不受影响） */
    document.addEventListener("touchmove", function (e) {
      if (e.touches && e.touches.length > 1 && e.cancelable) e.preventDefault();
    }, { passive: false });
    /* 桌面预览：Ctrl + 滚轮也当缩放处理 */
    document.addEventListener("wheel", function (e) {
      if (e.ctrlKey && e.cancelable) e.preventDefault();
    }, { passive: false });
    /* 双击放大 */
    document.addEventListener("dblclick", function (e) {
      if (e.cancelable) e.preventDefault();
    }, { passive: false });
  })();

  /* ═══ 1. 主题 ═══════════════════════════════════════════════════ */
  var lastDarkSent;   /* 上一次 onTheme 的深浅值；切换时要重建 iframe 卡片 */
  function rgb(hex) {
    if (typeof hex !== "string") return null;
    var s = hex.trim().replace(/^#/, "");
    if (s.length === 3) s = s[0] + s[0] + s[1] + s[1] + s[2] + s[2];
    if (!/^[0-9a-fA-F]{6}$/.test(s)) return null;
    return [parseInt(s.slice(0, 2), 16), parseInt(s.slice(2, 4), 16), parseInt(s.slice(4, 6), 16)];
  }
  function rgba(c, a) { return "rgba(" + c[0] + "," + c[1] + "," + c[2] + "," + a + ")"; }

  window.App = window.App || {};

  /* 终末地只有一套冷黑配色：宿主传深还是浅，这里都收敛到同一组色。
     dark 必须报 true —— 卡片里那些沙箱 iframe 的文档底色跟着它走，
     报 false 的话深色主题里会冒出一条白底预览。 */
  var ENDF = {
    light: { page:"#0a0d0e", text:"#eef3ef", muted:"#89938f", surface:"#151a1b",
             bubble:"#202728", primary:"#f2f500", onPrimary:"#101313", dark:"true" },
    dark:  { page:"#0a0d0e", text:"#eef3ef", muted:"#89938f", surface:"#151a1b",
             bubble:"#202728", primary:"#f2f500", onPrimary:"#101313", dark:"true" }
  };
  window.App.onTheme = function (c) {
    if (!c) return;
    var pal = ENDF[String(c.dark) === "true" ? "dark" : "light"], o = {};
    for (var pk in c) o[pk] = c[pk];
    for (var pk2 in pal) o[pk2] = pal[pk2];
    c = o;
    var root = document.documentElement, st = root.style, changed = false;

    if (c.page)      { st.setProperty("--page", c.page); changed = true; }
    if (c.text)      { st.setProperty("--text", c.text); changed = true; }
    if (c.muted)     st.setProperty("--muted", c.muted);
    if (c.surface)   st.setProperty("--surface", c.surface);
    if (c.bubble)    st.setProperty("--bubble", c.bubble);
    if (c.primary)   st.setProperty("--primary", c.primary);
    if (c.onPrimary) st.setProperty("--onPrimary", c.onPrimary);
    if (c.fontScale) st.setProperty("--fs", String(c.fontScale));

    var dark = String(c.dark) === "true";
    root.setAttribute("data-dark", dark ? "1" : "0");

    /* 只做透明度派生，色相一律用宿主给的最终色 */
    var t = rgb(c.text) || [238, 243, 239];
    st.setProperty("--hair", rgba(t, dark ? 0.16 : 0.10));
    st.setProperty("--veil", rgba(t, dark ? 0.07 : 0.045));

    /* 页面底色也写到 html 上，避免键盘顶起时露白 */
    if (c.page) {
      root.style.backgroundColor = c.page;
      /* 玻璃输入框的半透明底：压在正文上，太透会看不清字 */
      var pc = rgb(c.page);
      if (pc) st.setProperty("--composer-bg", rgba(pc, dark ? 0.62 : 0.74));
    }
    var sc = rgb(c.surface) || rgb(c.page);
    if (sc) {
      /* 现在真的有 backdrop-filter 了：底色必须更透（不然模糊等于没开），
         但也不能太透 —— 深色下 0.55 保住对比度，浅色下 0.66 不发灰 */
      st.setProperty("--glass-bg", rgba(sc, dark ? 0.55 : 0.66));
      st.setProperty("--glass-line", rgba(t, dark ? 0.22 : 0.13));
    }

    /* 深浅切换：**不要**整树重画（那会让整页闪一下白）。
       静态预览没有脚本，只能重建（新 iframe 淡入遮住白闪）；
       动态卡片（card:html / mermaid）发消息就地换色，连滚动位置都不动。 */
    if (typeof lastDarkSent === "boolean" && dark !== lastDarkSent) {
      retintStaticPreviews();
      var lives = listEl.querySelectorAll(".pv.live iframe");
      for (var li = 0; li < lives.length; li++) {
        try {
          lives[li].contentWindow.postMessage(
            { mojianTheme: 1, dark: dark }, "*");
        } catch (e) {}
      }
    }
    lastDarkSent = dark;

    if (changed) autosize();
  };

  /* ═══ 2. 文字标记：原文 + 第几次出现 → 字符区间 ═════════════════ */
  function marksSig(marks) {
    if (!marks || !marks.length) return "";
    var s = "";
    for (var i = 0; i < marks.length; i++) {
      var k = marks[i] || {};
      s += (k.text || "") + "~" + (k.occurrence | 0) + "~" + (k.color || "") + "~" + (k.kind || "") + ";";
    }
    return s;
  }

  /* 返回 [{s,e,color,kind}]，按位置排序并去掉重叠 */
  function markRanges(content, marks) {
    if (!marks || !marks.length || !content) return null;
    var out = [];
    for (var i = 0; i < marks.length; i++) {
      var mk = marks[i] || {};
      var txt = mk.text;
      if (!txt) continue;
      var occ = mk.occurrence | 0, idx = -1, from = 0;
      for (var n = 0; n <= occ; n++) {
        idx = content.indexOf(txt, from);
        if (idx < 0) break;
        from = idx + txt.length;
      }
      if (idx < 0) continue;                     /* 内容变了，标不上就跳过 */
      out.push({
        s: idx, e: idx + txt.length,
        kind: mk.kind === "quote" ? "q" : "m",
        color: mk.color || "yellow"
      });
    }
    if (!out.length) return null;
    out.sort(function (a, b) { return a.s - b.s || a.e - b.e; });
    var clean = [], lastEnd = -1;
    for (var j = 0; j < out.length; j++) {
      if (out[j].s >= lastEnd) { clean.push(out[j]); lastEnd = out[j].e; }
    }
    return clean;
  }

  /* 把一段纯文本写进 parent，按标记切开 */
  function emitText(parent, str, abs, ranges) {
    if (!ranges) { parent.appendChild(document.createTextNode(str)); return; }
    var pos = 0, len = str.length;
    for (var k = 0; k < ranges.length; k++) {
      var r = ranges[k];
      var e0 = r.e - abs;
      if (e0 <= pos) continue;
      var s0 = Math.max(r.s - abs, pos);
      if (s0 >= len) break;
      var e1 = Math.min(e0, len);
      if (s0 > pos) parent.appendChild(document.createTextNode(str.slice(pos, s0)));
      var mk = document.createElement("mark");
      mk.className = "mk " + (r.kind === "q" ? "q " : "") + r.color;
      mk.textContent = str.slice(s0, e1);
      parent.appendChild(mk);
      pos = e1;
    }
    if (pos < len) parent.appendChild(document.createTextNode(str.slice(pos)));
  }

  /* ═══ 3. 行内 Markdown ══════════════════════════════════════════ */
  /* 注意：递归时不能共用带 g 的正则实例 —— lastIndex 会被内层改写，
     外层 exec 会原地打转（曾经因此把 WebView 渲染进程卡死）。 */
  var INLINE_SRC = "(!\\[[^\\]\\n]*\\]\\([^)\\s]+\\))|(`[^`\\n]+`)|(\\*\\*[^*\\n]+\\*\\*)|(~~[^~\\n]+~~)|(\\[[^\\]\\n]+\\]\\([^)\\s]+\\))|(\\*[^*\\n]+\\*)|(\\$[^$\\s][^$\\n]*[^$\\s]\\$|\\$[^$\\s]\\$)";
  var SAFE_SCHEME = /^(https?:|mailto:|#)/i;
  var NET_IMG = /^(https:\/\/|data:image\/|blob:)/i;

  function mdImage(tok) {
    var mm = /!\[([^\]]*)\]\(([^)]+)\)$/.exec(tok);
    var alt = mm ? mm[1] : "";
    var src = mm ? mm[2] : "";
    var box = el("span", "mdimg");
    if (NET_IMG.test(src)) {
      var img = document.createElement("img");
      img.loading = "lazy";
      img.decoding = "async";
      img.alt = alt || "";
      img.setAttribute("data-act", "zoom");
      img.dataset.src = src;
      img.onerror = function () {
        box.textContent = "";
        box.appendChild(el("span", "nofs", "🖼 图片加载失败 · 点击复制地址"));
        box.setAttribute("data-act", "copyurl");
        box.dataset.url = src;
      };
      img.src = src;
      box.appendChild(img);
      if (alt) box.appendChild(el("span", "mdcap", alt));
    } else {
      /* 本地路径：走 App.image 换 data URI（§8.1）。读不到再给复制出口 */
      box.appendChild(el("span", "nofs", "🖼 图片读取中…"));
      resolveImage(src, function (data) {
        if (data) {
          box.textContent = "";
          var img = document.createElement("img");
          img.decoding = "async";
          img.alt = alt || "";
          img.setAttribute("data-act", "zoom");
          img.dataset.src = data;
          img.src = data;
          box.appendChild(img);
          if (alt) box.appendChild(el("span", "mdcap", alt));
        } else {
          box.textContent = "";
          box.appendChild(el("span", "nofs", "🖼 本地图片读不到 · 点击复制路径"));
          box.setAttribute("data-act", "copyfile");
          box.dataset.path = src;
        }
      });
    }
    return box;
  }

  function inline(parent, src, abs, ranges) {
    var re = new RegExp(INLINE_SRC, "g");
    var last = 0, m;
    while ((m = re.exec(src)) !== null) {
      if (m.index > last) emitText(parent, src.slice(last, m.index), abs + last, ranges);
      var tok = m[0];
      if (m[1]) {
        parent.appendChild(mdImage(tok));
      } else if (m[2]) {
        var cd = document.createElement("code");
        cd.textContent = tok.slice(1, -1);
        parent.appendChild(cd);
      } else if (m[3]) {
        var b = document.createElement("strong");
        inline(b, tok.slice(2, -2), abs + m.index + 2, ranges);
        parent.appendChild(b);
      } else if (m[4]) {
        var s = document.createElement("s");
        s.textContent = tok.slice(2, -2);
        parent.appendChild(s);
      } else if (m[5]) {
        var mm = /^\[([^\]]+)\]\(([^)]+)\)$/.exec(tok);
        if (mm && SAFE_SCHEME.test(mm[2])) {
          var a = document.createElement("a");
          a.textContent = mm[1];
          a.setAttribute("href", mm[2]);
          a.setAttribute("target", "_blank");
          a.setAttribute("rel", "noopener noreferrer");
          parent.appendChild(a);
        } else {
          emitText(parent, mm ? mm[1] : tok, abs + m.index, ranges);
        }
      } else if (m[6]) {
        var em = document.createElement("em");
        inline(em, tok.slice(1, -1), abs + m.index + 1, ranges);
        parent.appendChild(em);
      } else if (m[7]) {
        /* 行内公式 $...$；渲染不动时 mathRender 自己会降级成原文 */
        parent.appendChild(mathRender(tok.slice(1, -1), false));
      }
      last = m.index + tok.length;
    }
    if (last < src.length) emitText(parent, src.slice(last), abs + last, ranges);
  }

  function isBlockStart(line) {
    return /^\s*(```|#{1,6}\s|>|([-*+]|\d+[.)])\s+|([-*_])(\s*\3){2,}\s*$)/.test(line);
  }
  function splitRow(line) {
    var s = line.trim();
    if (s.charAt(0) === "|") s = s.slice(1);
    if (s.charAt(s.length - 1) === "|") s = s.slice(0, -1);
    var out = [], cell = "", esc = false;
    for (var k = 0; k < s.length; k++) {
      var ch = s.charAt(k);
      if (esc) { cell += ch; esc = false; continue; }
      if (ch === "\\") { esc = true; continue; }
      if (ch === "|") { out.push(cell.trim()); cell = ""; continue; }
      cell += ch;
    }
    out.push(cell.trim());
    return out;
  }

  /* 代码卡片：按语言分成 普通 / HTML 预览 / JS 运行 / 思维导图 /
     card:mermaid / card:image / card:html 七种形态。
     所有 iframe 都跑在 sandbox 里，并被 CSP 掐断 ——
     沙箱里唯一放行的是国内可达 CDN（npmmirror / jsdelivr）。 */
  var PAGE_CSP = "<meta http-equiv=\"Content-Security-Policy\" content=\"default-src 'none'; " +
    "style-src 'unsafe-inline'; img-src data: blob:; font-src data:\">";
  /* js 卡要**真的执行代码**，必须显式写 script-src：
     PAGE_CSP 里没有这一项，`default-src 'none'` 会把它一并封死 ——
     于是 iframe 里的脚本（连 bridge 一起）根本不跑，表现就是
     "点了运行、卡片里一行输出都没有、alert 也不弹"。 */
  var JS_DOC_CSP = "<meta http-equiv=\"Content-Security-Policy\" content=\"default-src 'none'; " +
    "script-src 'unsafe-inline'; style-src 'unsafe-inline'; " +
    "img-src data: blob: https:; media-src data: blob: https:; " +
    "connect-src https: data: blob:; font-src data:\">";
  var CDN_HOSTS = "https://registry.npmmirror.com https://fastly.jsdelivr.net https://cdn.jsdelivr.net";
  var MER_CDN = [
    "https://registry.npmmirror.com/mermaid/11.4.1/files/dist/mermaid.min.js",
    "https://fastly.jsdelivr.net/npm/mermaid@11.4.1/dist/mermaid.min.js",
    "https://cdn.jsdelivr.net/npm/mermaid@11.4.1/dist/mermaid.min.js"
  ];

  /* 沙箱卡片里"换主题不改 srcdoc、只换颜色"的钩子：
     重建 iframe 会闪一下白，动态卡片干脆用 postMessage 就地换色。 */
  var LIVE_THEME_HOOK =
    "window.addEventListener('message',function(ev){var d=ev.data;if(!d||!d.mojianTheme)return;" +
    "var dk=!!d.dark;" +
    "var st=document.getElementById('mj-th');" +
    "if(!st){st=document.createElement('style');st.id='mj-th';document.head.appendChild(st)}" +
    "st.textContent='html,body{background:transparent;color:'+(dk?'#E7E5DF':'#111')+'}'+" +
    "'body *{border-color:'+(dk?'rgba(231,229,223,.22)':'rgba(0,0,0,.12)')+'}'+" +
    "'::-webkit-scrollbar{width:0}';" +
    "if(window.mermaid&&window.__mmCode){try{" +
    "mermaid.initialize({startOnLoad:false,theme:dk?'dark':'default',securityLevel:'strict'});" +
    "mermaid.render('g'+Date.now(),window.__mmCode).then(function(r){" +
    "var o=document.getElementById('out');if(!o)return;o.innerHTML=r.svg;" +
    "var sv=o.querySelector('svg');if(sv){sv.style.maxWidth='100%';sv.style.height='auto'}" +
    "try{parent.postMessage({mojianCard:1,cid:CID,type:'height',h:Math.ceil(document.body.scrollHeight)+16},'*')}catch(e){}" +
    "}).catch(function(e){})}catch(e){}}});";
  var cardSeq = 0;

  function cardKind(lang) {
    var l = String(lang || "").toLowerCase();
    if (l === "card:mermaid" || l === "mermaid") return "mermaid";
    if (l === "card:image" || l === "image" || l === "img") return "image";
    if (l === "card:html") return "htmlcard";
    if (l === "mindmap" || l === "mm" || l === "mind-map") return "map";
    if (l === "js" || l === "javascript" || l === "mjs") return "js";
    if (l === "html" || l === "htm" || l === "svg" || l === "xml") return "html";
    return "plain";
  }

  function codeCard(lang, code, pending) {
    var kind = cardKind(lang);
    var fig = document.createElement("figure");
    fig.className = "codecard";
    fig.setAttribute("data-kind", kind);

    var cap = document.createElement("div");
    cap.className = "cap";
    var lg = document.createElement("span");
    lg.className = "lang";
    lg.textContent = lang || "code";
    cap.appendChild(lg);
    var grow = document.createElement("span");
    grow.className = "grow";
    cap.appendChild(grow);

    function btn(act, text) {
      var b = document.createElement("button");
      b.className = "mini";
      b.type = "button";
      b.setAttribute("data-act", act);
      b.textContent = text;
      cap.appendChild(b);
      return b;
    }

    var runBtn = null;
    if (kind === "js") runBtn = btn("run", "运行");
    else if (kind === "html") { runBtn = btn("run", "运行"); btn("src", "源码"); }
    else if (kind !== "plain" && kind !== "image") btn("src", "源码");
    btn("copy", "复制");

    var pre = document.createElement("pre");
    var cd = document.createElement("code");
    cd.textContent = code;
    pre.appendChild(cd);

    fig.appendChild(cap);
    fig.appendChild(pre);

    /* 流式期间代码还在变，预览/运行一律等收尾再说 */
    if (!pending && code.trim()) {
      if (kind === "html") buildPreview(fig, code);
      else if (kind === "htmlcard") buildLive(fig, code, "html");
      else if (kind === "mermaid") buildLive(fig, code, "mermaid");
      else if (kind === "image") buildImage(fig, code);
      else if (kind === "map") buildMap(fig, code);
      else if (kind === "js" && runBtn) runJs(fig, code, runBtn);   /* 默认就跑 */
    }
    return fig;
  }

  /* HTML / SVG 预览：沙箱 iframe，禁脚本、禁网络 */
  function buildPreview(fig, code) {
    var box = document.createElement("div");
    box.className = "pv";
    var bar = document.createElement("div");
    bar.className = "pbar";
    bar.appendChild(document.createElement("span")).textContent = "预览 · 脚本与网络已禁用";
    box.appendChild(bar);
    var ifr = document.createElement("iframe");
    ifr.setAttribute("sandbox", "");
    ifr.setAttribute("referrerpolicy", "no-referrer");
    ifr.setAttribute("loading", "lazy");
    /* 底色透明，跟随主题 —— 写死 #fff 会在深色下白一块 */
    var dk = document.documentElement.getAttribute("data-dark") === "1";
    ifr.srcdoc = "<!DOCTYPE html>" + PAGE_CSP +
      "<meta name=\"color-scheme\" content=\"" + (dk ? "dark" : "light") + "\">" +
      "<style>html,body{margin:0;background:transparent;color:" + (dk ? "#E7E5DF" : "#111") + ";" +
      "font:14px/1.6 -apple-system,'PingFang SC',sans-serif}</style>" + code;
    /* 载入完成才显形：新 iframe 第一帧是白的，淡入可以盖掉那次闪白 */
    ifr.addEventListener("load", function () { ifr.classList.add("rdy"); });
    box.appendChild(ifr);
    fig.appendChild(box);

    /* 静态预览的底色是构建期写死的，换主题时要重建 —— 记下来 */
    fig.__pvCode = code;
    if (staticPvs.indexOf(fig) < 0) staticPvs.push(fig);
  }

  /* 换深浅色时重建静态预览（它们没有脚本，改不了自己的颜色） */
  var staticPvs = [];
  function retintStaticPreviews() {
    for (var i = staticPvs.length - 1; i >= 0; i--) {
      var fig = staticPvs[i];
      if (!fig.isConnected) { staticPvs.splice(i, 1); continue; }
      var old = fig.querySelector(".pv");
      if (old) old.remove();
      buildPreview(fig, fig.__pvCode);
    }
  }

  /* 思维导图：吃 mermaid mindmap / 纯缩进列表，画成一棵横向树 */
  function fitText(text, maxW, fs) {
    var w = 0, out = "";
    for (var i = 0; i < text.length; i++) {
      var cw = text.charCodeAt(i) > 255 ? fs : fs * 0.56;
      if (w + cw > maxW) { out += "…"; break; }
      w += cw;
      out += text[i];
    }
    return out;
  }

  function parseMind(code) {
    var lines = String(code || "").split("\n");
    var root = null, stack = [], count = 0;
    for (var i = 0; i < lines.length; i++) {
      var raw = lines[i];
      if (!raw.trim() || /^\s*mindmap\s*$/i.test(raw)) continue;
      if (/^\s*%%/.test(raw)) continue;
      var indent = raw.match(/^\s*/)[0].replace(/\t/g, "  ").length;
      var text = raw.trim();
      /* 去掉 mermaid 的形状语法：root((文本)) / id[文本] / id{文本} */
      var mm = /^(?:[\w.\-]+\s*)?(\(\(|\[\[|\{\{|\(|\[|\{)(.*?)(\)\)|\]\]|\}\}|\)|\]|\})$/.exec(text);
      if (mm) text = mm[2];
      if (!text) continue;
      if (++count > 200) break;

      while (stack.length && indent <= stack[stack.length - 1].indent) stack.pop();
      var node = { text: text, children: [] };
      if (!stack.length) {
        if (!root) root = node;
        else root.children.push(node);      /* 后面出现同级根，挂到第一根下 */
      } else {
        stack[stack.length - 1].node.children.push(node);
      }
      stack.push({ indent: indent, node: node });
    }
    return root;
  }

  function buildMap(fig, code) {
    var root = parseMind(code);
    var box = document.createElement("div");
    box.className = "map";
    if (!root) {
      box.textContent = "";
      box.style.cssText += "padding:12px;color:var(--muted);font:12px/1.5 var(--sans)";
      box.textContent = "（没有解析出节点）";
      fig.appendChild(box);
      return;
    }

    var FS = 12, ROWH = 27, GAPY = 7, GAPX = 30, PAD = 10;
    var levelW = [], levelX = [];

    (function measure(n, d) {
      var w = Math.min(240, Math.max(64, n.text.length * FS * 1.08 + 24));
      n.w = w;
      levelW[d] = Math.max(levelW[d] || 0, w);
      for (var i = 0; i < n.children.length; i++) measure(n.children[i], d + 1);
    })(root, 0);

    var acc = PAD;
    for (var d = 0; d < levelW.length; d++) { levelX[d] = acc; acc += levelW[d] + GAPX; }
    var totalW = acc - GAPX + PAD;

    var cursor = PAD;
    (function place(n, d) {
      n.x = levelX[d];
      n.d = d;
      if (!n.children.length) {
        n.y = cursor + ROWH / 2;
        cursor += ROWH + GAPY;
      } else {
        for (var i = 0; i < n.children.length; i++) place(n.children[i], d + 1);
        n.y = (n.children[0].y + n.children[n.children.length - 1].y) / 2;
      }
    })(root, 0);
    var totalH = cursor - GAPY + PAD;

    var NS = "http://www.w3.org/2000/svg";
    var svg = document.createElementNS(NS, "svg");
    svg.setAttribute("width", Math.round(totalW));
    svg.setAttribute("height", Math.round(totalH));
    svg.setAttribute("viewBox", "0 0 " + Math.round(totalW) + " " + Math.round(totalH));

    /* 先线后节点，避免连线压住圆角 */
    (function links(n) {
      for (var i = 0; i < n.children.length; i++) {
        var c = n.children[i];
        var x1 = n.x + n.w, y1 = n.y, x2 = c.x, y2 = c.y;
        var mx = (x1 + x2) / 2;
        var p = document.createElementNS(NS, "path");
        p.setAttribute("class", "mn-link");
        p.setAttribute("d", "M" + x1 + "," + y1 + " C" + mx + "," + y1 + " " + mx + "," + y2 + " " + x2 + "," + y2);
        svg.appendChild(p);
        links(c);
      }
    })(root);

    (function draw(n) {
      var rect = document.createElementNS(NS, "rect");
      rect.setAttribute("x", n.x);
      rect.setAttribute("y", n.y - ROWH / 2);
      rect.setAttribute("width", n.w);
      rect.setAttribute("height", ROWH);
      rect.setAttribute("rx", n.d === 0 ? 9 : 7);
      rect.setAttribute("class", n.d === 0 ? "mn-rect mn-root" : "mn-rect");
      svg.appendChild(rect);

      var t = document.createElementNS(NS, "text");
      t.setAttribute("x", n.x + 10);
      t.setAttribute("y", n.y + 4);
      t.setAttribute("class", n.d === 0 ? "mn-t mn-t-root" : "mn-t");
      t.textContent = fitText(n.text, n.w - 18, FS);
      svg.appendChild(t);

      for (var i = 0; i < n.children.length; i++) draw(n.children[i]);
    })(root);

    box.appendChild(svg);
    fig.appendChild(box);
  }

  /* ── card:mermaid / card:html：沙箱里跑动态内容 ─────────────────
     sandbox=allow-scripts（无同源），CSP 只放行国内可达 CDN。
     渲染完用 postMessage 把高度送回来，卡片随内容长。 */
  function buildLive(fig, code, mode) {
    var cid = "mc" + (++cardSeq);
    var dark = document.documentElement.getAttribute("data-dark") === "1";
    var box = el("div", "pv live");
    var bar = el("div", "pbar");
    bar.appendChild(el("span", null,
      mode === "mermaid" ? "mermaid · 沙箱渲染" : "交互卡片 · 沙箱运行"));
    box.appendChild(bar);
    var ifr = document.createElement("iframe");
    ifr.dataset.cid = cid;
    ifr.setAttribute("sandbox", "allow-scripts");
    ifr.setAttribute("referrerpolicy", "no-referrer");
    ifr.setAttribute("loading", "lazy");
    /* 音频/视频要能被用户点开播：不委托 autoplay 的话，卡片里 play()
       仍然会被"需要用户手势"拦掉（宿主侧 mediaPlaybackRequiresUserGesture
       走的是同一套策略）。 */
    ifr.setAttribute("allow", "autoplay; fullscreen; encrypted-media; picture-in-picture");
    ifr.srcdoc = liveDoc(mode, code, cid, dark);
    ifr.addEventListener("load", function () { ifr.classList.add("rdy"); });
    box.appendChild(ifr);
    fig.appendChild(box);
    fig.__liveMode = mode;
  }

  function liveDoc(mode, code, cid, dark) {
    /* CSP 白名单。注意 `media-src` 和 `connect-src` 必须显式写：
       开了 `default-src 'none'` 之后，没声明的指令默认就是 none ——
       于是卡片里的 <audio>/<video> 一律加载失败，报的是
       MEDIA_ERR_SRC_NOT_SUPPORTED（error.code 4），看起来像"地址 404"，
       其实一次请求都没发出去。音乐/视频卡就是这么"404"的。
       fetch/XHR 同理（天气、汇率这类要拉接口的卡片）。 */
    var net = mode === "html";
    var csp = "<meta http-equiv=\"Content-Security-Policy\" content=\"default-src 'none'; " +
      "script-src 'unsafe-inline' " + CDN_HOSTS + "; " +
      "style-src 'unsafe-inline' " + CDN_HOSTS + "; " +
      "img-src data: blob:" + (net ? " https:" : "") + "; " +
      "media-src data: blob:" + (net ? " https:" : "") + "; " +
      "connect-src " + (net ? "'self' https: data: blob:" : "'none'") + "; " +
      "font-src data: " + CDN_HOSTS + "\">";
    var head = "<!DOCTYPE html>" + csp +
      "<meta name=\"viewport\" content=\"width=device-width,initial-scale=1\">" +
      "<meta name=\"color-scheme\" content=\"" + (dark ? "dark" : "light") + "\">" +
      "<style>*{box-sizing:border-box}html,body{margin:0;background:transparent;color:" +
      (dark ? "#E7E5DF" : "#111") + ";" +
      "font:14px/1.6 -apple-system,'PingFang SC',sans-serif;overflow-x:hidden}" +
      ".wrap{padding:10px;min-height:20px}</style>" + JS_BRIDGE;
    var body;

    if (mode === "mermaid") {
      body = "<div class=\"wrap\" id=\"out\"><div class=\"ld\">正在加载 mermaid…</div></div>" +
        "<scr" + "ipt>var CDNS=" + JSON.stringify(MER_CDN) + ";" +
        "var CODE=" + JSON.stringify(code).replace(/<\//g, "<\\\\/") + ";" +
        "window.__mmCode=CODE;" +
        "var DARK=" + (dark ? "true" : "false") + ";" +
        "var CID=" + JSON.stringify(cid) + ";" +
        "function post(t,x){try{parent.postMessage({mojianCard:1,cid:CID,type:t,h:x},'*')}catch(e){}}" +
        "function fail(m){var o=document.getElementById('out');o.textContent=m;" +
        "post('err');post('height',60)}" +
        "function load(i){if(i>=CDNS.length){fail('mermaid CDN 全部不可达');return}" +
        "var s=document.createElement('script');s.src=CDNS[i];" +
        "s.onload=function(){try{init()}catch(e){fail(String(e))}};" +
        "s.onerror=function(){load(i+1)};document.head.appendChild(s)}" +
        "function init(){mermaid.initialize({startOnLoad:false,theme:DARK?'dark':'default',securityLevel:'strict'});" +
        "mermaid.render('g'+Date.now(),CODE).then(function(r){" +
        "var o=document.getElementById('out');o.innerHTML=r.svg;" +
        "var sv=o.querySelector('svg');if(sv){sv.style.maxWidth='100%';sv.style.height='auto'}" +
        "requestAnimationFrame(function(){post('height',Math.ceil(document.body.scrollHeight)+16)})" +
        "}).catch(function(e){fail(String(e&&e.message||e))})}" +
        "load(0);" + LIVE_THEME_HOOK + "</scr" + "ipt>";
    } else {
      body = "<div class=\"wrap\" id=\"rt\">" + code + "</div>" +
        "<scr" + "ipt>var CID=" + JSON.stringify(cid) + ";" +
        "function post(h){try{parent.postMessage({mojianCard:1,cid:CID,type:'height',h:h},'*')}catch(e){}}" +
        "function ping(){post(Math.ceil(document.body.scrollHeight)+10)}" +
        "if(window.ResizeObserver){new ResizeObserver(ping).observe(document.getElementById('rt'))}" +
        "window.addEventListener('load',function(){setTimeout(ping,80)});ping();" +
        LIVE_THEME_HOOK + "</scr" + "ipt>";
    }
    return head + body;
  }

  /* ── card:image：远程 URL 直接显示；本地路径走 App.image 换 data URI ── */
  function buildImage(fig, code) {
    var src = code.trim().split("\n")[0].trim();
    var box = el("div", "pv imgc");
    if (NET_IMG.test(src)) {
      var img = document.createElement("img");
      img.loading = "lazy";
      img.decoding = "async";
      img.setAttribute("data-act", "zoom");
      img.dataset.src = src;
      img.onerror = function () {
        box.textContent = "";
        box.appendChild(el("div", "nofs2", "图片加载失败 · 点击复制地址"));
        box.setAttribute("data-act", "copyurl");
        box.dataset.url = src;
      };
      img.src = src;
      box.appendChild(img);
    } else {
      box.appendChild(el("div", "nofs2", "图片读取中…"));
      resolveImage(src, function (data) {
        if (data) {
          box.textContent = "";
          var img2 = document.createElement("img");
          img2.decoding = "async";
          img2.setAttribute("data-act", "zoom");
          img2.dataset.src = data;
          img2.src = data;
          box.appendChild(img2);
        } else {
          box.textContent = "";
          box.appendChild(el("div", "nofs2", "本地图片读不到 · 点击复制路径"));
          box.setAttribute("data-act", "copyfile");
          box.dataset.path = src;
        }
      });
    }
    fig.appendChild(box);
  }

  /* ── 正文里的链接 / 本地文件：自动补成小卡片 ──────────────────── */
  var AUTOLINK_RE = /(https?:\/\/[^\s<>，。；、）】」"'！]{4,})|((?:[A-Za-z]:[\\/][^\s<>，。；、）】」"'！]{2,})|(?:\/(?:sdcard|storage|data|home|Users|mnt)\/[^\s<>，。；、）】」"'！]{2,}))/g;

  function linkChip(url) {
    var d = url, mm = /^https?:\/\/([^\/\s]+)/i.exec(url);
    if (mm) d = mm[1].replace(/^www\./, "");
    var isApi = /\/api(\/|$)|\.json($|\?)|\/v\d+(\/|$)/i.test(url);
    var c = el("span", "lchip");
    c.setAttribute("data-act", "copyurl");
    c.dataset.url = url;
    c.appendChild(el("b", null, d));
    c.appendChild(el("span", "tagx", isApi ? "API" : "网页"));
    return c;
  }
  function fileChip(path) {
    var name = path.split(/[\\/]/).pop() || path;
    var c = el("span", "fchip");
    /* 点一下就地看内容（文本走宿主 readFile，其它交给系统打开）；
       "复制路径"移到预览页的「复制」和长按菜单里 */
    c.setAttribute("data-act", "openfile");
    c.dataset.path = path;
    c.appendChild(el("b", null, "📄 " + name));
    c.appendChild(el("span", "tagx", isTextPath(path) ? "查看" : "打开"));
    return c;
  }
  /* 把纯文本节点里的裸链接 / 本地路径换成小卡片（不碰代码和已有链接） */
  function autolink(root) {
    var tw = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, null, false);
    var texts = [], t;
    while ((t = tw.nextNode())) {
      var p = t.parentNode;
      var tag = p && p.nodeName;
      if (tag === "CODE" || tag === "A" || tag === "PRE" || tag === "MARK") continue;
      /* 附件芯片里的文件名不该再被自动识别成"路径卡片" */
      if (p && p.closest && p.closest(".meatts, .fchip, .lchip")) continue;
      texts.push(t);
    }
    for (var i = 0; i < texts.length; i++) {
      var node = texts[i], s = node.nodeValue;
      if (!s || s.length < 6) continue;
      AUTOLINK_RE.lastIndex = 0;
      if (!AUTOLINK_RE.test(s)) continue;
      AUTOLINK_RE.lastIndex = 0;
      var frag = document.createDocumentFragment(), last = 0, m;
      while ((m = AUTOLINK_RE.exec(s))) {
        if (m.index > last) frag.appendChild(document.createTextNode(s.slice(last, m.index)));
        frag.appendChild(m[1] ? linkChip(m[1]) : fileChip(m[2]));
        last = m.index + m[0].length;
      }
      if (last < s.length) frag.appendChild(document.createTextNode(s.slice(last)));
      node.parentNode.replaceChild(frag, node);
    }
  }

  /* 点开图片全屏 */
  function zoomImg(src, alt) {
    if (!src) return;
    var ov = el("div", "zoom");
    var img = document.createElement("img");
    img.src = src;
    ov.appendChild(img);
    if (alt) ov.appendChild(el("div", "cap", alt));
    ov.appendChild(el("div", "cap", "点击任意处关闭"));
    ov.addEventListener("click", function () { ov.remove(); });
    document.body.appendChild(ov);
  }

  /* ── 本地文件：宿主代读 → 页内摊开 ──────────────────────────────
     页面自己读不了盘（沙箱），内容由宿主的 readFile handler 提供：
     白名单 + 4MB 上限 + 二进制识别。这里只负责展示，
     以及"太大 / 二进制"时把人引到「用系统打开」。 */
  var TEXT_EXT = /\.(md|markdown|mdx|txt|text|json|jsonl|csv|tsv|log|xml|yml|yaml|ini|conf|cfg|toml|js|mjs|cjs|ts|jsx|tsx|css|scss|sass|less|html|htm|vue|svelte|py|dart|java|kt|kts|c|h|cc|cpp|hpp|cs|go|rs|rb|php|sh|bash|zsh|fish|sql|swift|lua|r|pl|gradle|properties|env)$/i;

  function isTextPath(p) {
    var s = String(p || "").split("?")[0];
    var name = s.split(/[\\/]/).pop() || s;
    if (/^(dockerfile|makefile|readme|license|gemfile)$/i.test(name)) return true;
    return TEXT_EXT.test(name);
  }
  function readTextFile(path, limit) {
    if (!HAS_BRIDGE) {
      return Promise.resolve({ ok: false, error: "预览模式读不了本地文件" });
    }
    var p = null;
    try {
      p = (window.App && typeof window.App.readFile === "function")
        ? window.App.readFile(path, limit || 0)
        : call("readFile", path, limit || 0);
    } catch (e) { p = null; }
    return Promise.resolve(p).then(function (raw) {
      if (raw && typeof raw === "object") return raw;
      if (!raw) return { ok: false, error: "宿主没有返回内容" };
      try { return JSON.parse(raw) || { ok: false, error: "宿主返回空" }; }
      catch (e) { return { ok: false, error: "宿主返回的内容无法解析" }; }
    }, function () { return { ok: false, error: "读取失败" }; });
  }
  function fmtSize(n) {
    n = n | 0;
    if (n < 1024) return n + " B";
    if (n < 1048576) return (n / 1024).toFixed(1) + " KB";
    return (n / 1048576).toFixed(1) + " MB";
  }
  function previewFile(path, label) {
    var p = String(path || "");
    if (!p) { toast("没有路径"); return; }
    var name = label || p.split(/[\\/]/).pop() || p;

    var ov = el("div", "fview");
    var head = el("div", "fhead");
    head.appendChild(el("div", "fname", name));
    var meta = el("span", "fmeta", "读取中…");
    head.appendChild(meta);

    var btnCopy = el("button", null, "复制");
    btnCopy.addEventListener("click", function () {
      var t = ov.querySelector("pre");
      if (t) copyPlain(t.textContent, "内容已复制");
      else copyPlain(p, "路径已复制");
    });
    var btnOpen = el("button", null, "打开");
    btnOpen.addEventListener("click", function () {
      if (!HAS_BRIDGE) { toast("预览模式没有宿主"); return; }
      call("open", p);
    });
    var btnX = el("button", null, "关闭");
    btnX.addEventListener("click", function () { ov.remove(); });
    head.appendChild(btnCopy);
    head.appendChild(btnOpen);
    head.appendChild(btnX);

    var body = el("div", "fbody");
    ov.appendChild(head);
    ov.appendChild(body);
    document.body.appendChild(ov);

    readTextFile(p, 300000).then(function (r) {
      body.textContent = "";
      if (!r || !r.ok) {
        meta.textContent = "";
        body.appendChild(el("p", "fnote", (r && r.error) || "读不到这个文件"));
        body.appendChild(el("p", "fnote", "可以点右上角「打开」，交给系统里的应用。"));
        return;
      }
      meta.textContent = fmtSize(r.size) + (r.truncated ? " · 已截断" : "");
      if (r.binary) {
        body.appendChild(el("p", "fnote", "这是二进制文件，页面上摊不开。"));
        body.appendChild(el("p", "fnote", "点右上角「打开」用系统应用查看。"));
        return;
      }
      if (!r.text) {
        body.appendChild(el("p", "fnote", "文件是空的。"));
        return;
      }
      body.appendChild(el("pre", null, r.text));
      if (r.truncated) {
        body.appendChild(el("p", "fnote", "（内容过长，只显示前 30 万字符）"));
      }
    });
  }
  /* 文件类路径：文本就地摊开，其余交给系统 */
  function openLocalPath(p) {
    var path = String(p || "");
    if (!path) return;
    if (isTextPath(path)) { previewFile(path); return; }
    if (!HAS_BRIDGE) { toast("预览模式没有宿主"); return; }
    call("open", path);
  }

  /* JS 卡片：在 sandbox（allow-scripts 但**不同源**）里跑，
     用 postMessage 把 console 输出送回页面 */
  var JS_BRIDGE =
    "<scr" + "ipt>(function(){" +
    "function fmt(v){try{if(typeof v==='string')return v;" +
    "if(v instanceof Error)return v.stack||v.message;" +
    "return JSON.stringify(v)}catch(e){return String(v)}}" +
    "function post(t,a,toCard){var x=Array.prototype.map.call(a,fmt).join(' ');try{" +
    /* 在卡片里（不是 js 卡）：报错和提示塞回卡片底部那一行。
       CID 取不到就发空串，由父页面按消息来源反查是哪张卡 ——
       卡片脚本的 <script> 常常比 `var CID=…` 先执行。 */
    "if(toCard&&typeof __MJ_JS==='undefined'){" +
    "parent.postMessage({mojianCard:1,cid:(typeof CID!=='undefined'&&CID)||'',type:(t==='error'?'err':'note'),text:x},'*');return}" +
    "parent.postMessage({mojian:1,type:t,text:x},'*')" +
    "}catch(e){}}" +
    "['log','info','warn','error'].forEach(function(k){" +
    "var o=console[k];console[k]=function(){post(k,arguments);try{o&&o.apply(console,arguments)}catch(e){}};" +
    "});" +
    "window.onerror=function(m,u,l){post('error',[m+'  (第 '+l+' 行)'],1);return false};" +
    /* 未处理的 Promise 拒绝同样只吐一行，别让 WebView 弹出红色报错框 */
    "window.addEventListener('unhandledrejection',function(e){" +
    "var r=e&&e.reason;post('error',['Promise 未处理：'+" +
    "(r&&(r.message||r)||'未知原因')],1);e.preventDefault&&e.preventDefault()});" +
    /* alert/confirm/prompt 在 WebView 里会弹原生对话框（有的机型直接卡死渲染），
       一律改成"卡片里的一行字"：alert 当提示、confirm 返回 false、prompt 返回 null */
    "window.alert=function(){post('note',arguments,1);return undefined};" +
    "window.confirm=function(){post('note',['确认框已忽略：'+Array.prototype.join.call(arguments,' ')],1);return false};" +
    "window.prompt=function(){post('note',['输入框已忽略：'+Array.prototype.join.call(arguments,' ')],1);return null};" +
    "})();</scr" + "ipt>";

  function runJs(fig, code, btn) {
    var out = fig.querySelector(".jsout");
    if (!out) {
      out = document.createElement("div");
      out.className = "jsout";
      out.hidden = true;
      fig.appendChild(out);
    }
    /* 再点一次 = 停止：直接把 iframe 拆掉 */
    var old = fig.querySelector(".jsrun");
    if (old) {
      old.remove();
      if (btn) btn.textContent = "运行";
      if (btn) btn.classList.remove("done");
      return;
    }
    out.hidden = false;
    out.textContent = "";

    var ifr = document.createElement("iframe");
    ifr.className = "jsrun";
    ifr.setAttribute("sandbox", "allow-scripts");
    ifr.setAttribute("referrerpolicy", "no-referrer");
    ifr.__out = out;
    /* `__MJ_JS=1` 让 bridge 知道"这是纯执行环境，输出走 .jsout"，
       别把 console 当成卡片提示条 */
    ifr.srcdoc = "<!DOCTYPE html>" + JS_DOC_CSP +
      "<style>html,body{margin:0;background:transparent}</style>" +
      "<scr" + "ipt>var __MJ_JS=1</scr" + "ipt>" + JS_BRIDGE +
      "<scr" + "ipt>" + code + "</scr" + "ipt>";

    fig.appendChild(ifr);
    if (btn) { btn.textContent = "停止"; btn.classList.add("done"); }
    var line = document.createElement("span");
    line.className = "ln dim";
    line.textContent = "▶ 已运行（1000ms 后自动停止）";
    out.appendChild(line);

    /* 死循环保护：跑 1 秒就拆掉，别让页面陪着卡 */
    setTimeout(function () {
      if (ifr.parentNode) {
        ifr.remove();
        if (btn) { btn.textContent = "运行"; btn.classList.remove("done"); }
      }
    }, 1000);
  }

  /* 输出回流：js 卡的 console、动态卡片的自适应高度都从这里进来 */
  /* 卡片内的提示条：卡片脚本里的 alert / 报错都变成卡片底部的一行字。
     绝不调用 window.alert —— WebView 里那会拉起原生对话框，
     把正在滚动的页面卡住，看起来像"应用弹错了"。 */
  function cardNote(card, text, isErr) {
    if (!card || !text) return;
    var bar = card.querySelector(".cnote");
    if (!bar) {
      bar = el("div", "cnote");
      card.appendChild(bar);
    }
    var key = (isErr ? "e:" : "n:") + text;
    if (bar.__keys && bar.__keys.indexOf(key) >= 0) return;
    if (!bar.__keys) bar.__keys = [];
    bar.__keys.push(key);
    while (bar.__keys.length > 4) {
      bar.__keys.shift();
      if (bar.firstChild) bar.removeChild(bar.firstChild);
    }
    var row = el("span", "cnrow" + (isErr ? " bad" : ""));
    row.textContent = (isErr ? "✗ " : "ⓘ ") + text;
    bar.appendChild(row);
    bar.hidden = false;
    clearTimeout(bar.__t);
    if (!isErr) {
      bar.__t = setTimeout(function () {
        if (bar && bar.parentNode) bar.parentNode.removeChild(bar);
      }, 3600);
    }
  }

  window.addEventListener("message", function (e) {
    var d = e.data;
    if (!d) return;

    if (d.mojianCard === 1) {
      var fr = d.cid ? listEl.querySelector('iframe[data-cid="' + d.cid + '"]') : null;
      if (!fr) {
        /* 卡片脚本可能比 `var CID` 先执行，cid 是空的 —— 按消息来源反查 */
        var anyIf = listEl.querySelectorAll("iframe[data-cid]");
        for (var fi = 0; fi < anyIf.length; fi++) {
          if (anyIf[fi].contentWindow === e.source) { fr = anyIf[fi]; break; }
        }
      }
      if (!fr) return;
      if (d.type === "height") {
        fr.style.height = Math.max(60, Math.min(560, d.h | 0)) + "px";
      } else if (d.type === "err" && d.text) {
        /* 卡片里抛的错：贴在卡片底部一行字，不弹任何窗口 */
        cardNote(fr.closest(".codecard"), d.text, true);
      } else if (d.type === "note") {
        cardNote(fr.closest(".codecard"), d.text, false);
      } else if (d.type === "err") {
        var fg = fr.closest(".codecard");
        if (fg) fg.classList.add("showsrc");
      }
      return;
    }
    if (d.mojian !== 1) return;

    var frames = listEl.querySelectorAll(".jsrun");
    var src = null;
    for (var i = 0; i < frames.length; i++) {
      if (frames[i].contentWindow === e.source) { src = frames[i]; break; }
    }
    if (!src || !src.__out) return;
    var ln = document.createElement("span");
    ln.className = "ln" + (d.type === "error" ? " err" : (d.type === "note" ? " dim" : ""));
    ln.textContent = (d.type === "error" ? "✗ " : (d.type === "note" ? "ⓘ " : "› ")) + d.text;
    src.__out.appendChild(ln);
    src.__out.scrollTop = src.__out.scrollHeight;
  });

  /* 渲染一段 Markdown。
     opts.base  —— 这段文字在整条消息里的起始偏移（标记要用全局偏移）
     opts.ranges —— 标记区间
     返回 DocumentFragment；每个顶层块带 __end（该块结束处的全局偏移），
     流式增量渲染靠它判断"哪些块已经定稿"。 */
  /* ── 数学公式：LaTeX 常用子集，纯 DOM 自绘（不联网、不引库）────
     支持：上下标、分式、根号、大型算符（∑∏∫ 带上下限）、希腊字母、
     关系/箭头符号、\text、\mathbb、\hat/\bar/\vec、常用函数名。
     解析不动就原样输出文本 —— 公式坏了不能把消息渲染整个拖死。 */
  var GREEK = {alpha:"α",beta:"β",gamma:"γ",delta:"δ",epsilon:"ε",varepsilon:"ε",zeta:"ζ",eta:"η",theta:"θ",vartheta:"ϑ",iota:"ι",kappa:"κ",lambda:"λ",mu:"μ",nu:"ν",xi:"ξ",pi:"π",rho:"ρ",sigma:"σ",tau:"τ",upsilon:"υ",phi:"φ",varphi:"φ",chi:"χ",psi:"ψ",omega:"ω",Gamma:"Γ",Delta:"Δ",Theta:"Θ",Lambda:"Λ",Xi:"Ξ",Pi:"Π",Upsilon:"Υ",Sigma:"Σ",Phi:"Φ",Psi:"Ψ",Omega:"Ω"};
  var MSYM = {times:"×",cdot:"·",div:"÷",pm:"±",mp:"∓",leq:"≤",le:"≤",geq:"≥",ge:"≥",neq:"≠",ne:"≠",approx:"≈",equiv:"≡",sim:"∼",simeq:"≃",propto:"∝",infty:"∞",rightarrow:"→",to:"→",leftarrow:"←",gets:"←",leftrightarrow:"↔",Rightarrow:"⇒",Leftarrow:"⇐",Leftrightarrow:"⇔",mapsto:"↦",uparrow:"↑",downarrow:"↓",in:"∈",notin:"∉",ni:"∋",subset:"⊂",subseteq:"⊆",supset:"⊃",supseteq:"⊇",cup:"∪",cap:"∩",emptyset:"∅",varnothing:"∅",forall:"∀",exists:"∃",nexists:"∄",neg:"¬",lnot:"¬",land:"∧",lor:"∨",oplus:"⊕",otimes:"⊗",perp:"⊥",parallel:"∥",angle:"∠",degree:"°",prime:"′",hbar:"ℏ",ell:"ℓ",Re:"ℜ",Im:"ℑ",nabla:"∇",partial:"∂",dots:"…",ldots:"…",cdots:"⋯",vdots:"⋮",ddots:"⋱",langle:"⟨",rangle:"⟩","{":"{","}":"}","|":"‖","%":"%","&":"&","#":"#","_":"_","$":"$"};
  var BIGOPS = {sum:"∑",prod:"∏",coprod:"∐",int:"∫",iint:"∬",iiint:"∭",oint:"∮",bigcup:"⋃",bigcap:"⋂",bigoplus:"⊕",bigotimes:"⊗"};
  var MFUNCS = {sin:1,cos:1,tan:1,cot:1,sec:1,csc:1,arcsin:1,arccos:1,arctan:1,sinh:1,cosh:1,tanh:1,ln:1,log:1,exp:1,det:1,dim:1,gcd:1,max:1,min:1,sup:1,inf:1,Pr:1,arg:1};
  var MBB = {R:"ℝ",Z:"ℤ",N:"ℕ",Q:"ℚ",C:"ℂ",P:"ℙ",H:"ℍ",F:"𝔽",E:"𝔼"};
  var MACC = {hat:"\u0302",bar:"\u0305",vec:"\u20D7",tilde:"\u0303",dot:"\u0307",ddot:"\u0308",breve:"\u0306",check:"\u030C"};

  function mathNode(cls, txt) {
    var e = el("span", cls);
    if (txt != null) e.textContent = txt;
    return e;
  }
  /* 环境体按 \\ 断行、按 & 分列（都在花括号深度 0 上切） */
  function splitRows(s, a, b) {
    var rows = [], depth = 0, st = a;
    for (var i = a; i < b - 1; i++) {
      if (s[i] === "{") depth++;
      else if (s[i] === "}") depth--;
      else if (depth === 0 && s[i] === "\\" && s[i + 1] === "\\") {
        rows.push({ a: st, b: i }); st = i + 2; i++;
      }
    }
    rows.push({ a: st, b: b });
    return rows;
  }
  function splitCols(s, a, b) {
    var cols = [], depth = 0, st = a;
    for (var i = a; i < b; i++) {
      if (s[i] === "{") depth++;
      else if (s[i] === "}") depth--;
      else if (depth === 0 && s[i] === "&") { cols.push({ a: st, b: i }); st = i + 1; }
    }
    cols.push({ a: st, b: b });
    return cols;
  }

  function mathRender(tex, block) {
    var root = el("span", "math" + (block ? " blk" : ""));
    var pendingLim = null;
    try { parseSeq(root, tex, 0, tex.length); }
    catch (e) { root.textContent = tex; }
    return root;

    function matchBrace(s, k, b) {
      var depth = 0;
      for (var t = k; t < b; t++) {
        if (s[t] === "{") depth++;
        else if (s[t] === "}") { if (--depth === 0) return t; }
      }
      return -1;
    }
    function readArg(s, k, b) {
      while (k < b && /\s/.test(s[k])) k++;
      if (k >= b) return { a: b, b: b, next: b };
      if (s[k] === "{") {
        var d = matchBrace(s, k, b);
        if (d < 0) return { a: k + 1, b: b, next: b };
        return { a: k + 1, b: d, next: d + 1 };
      }
      if (s[k] === "\\") {
        var t2 = k + 1;
        while (t2 < b && /[a-zA-Z]/.test(s[t2])) t2++;
        if (t2 === k + 1) t2 = k + 2;
        return { a: k, b: t2, next: t2 };
      }
      return { a: k, b: k + 1, next: k + 1 };
    }
    function parseSupSub(out, s, i, b, which) {
      var tgt;
      if (pendingLim) { tgt = pendingLim; }
      else tgt = out;
      var r = readArg(s, i + 1, b);
      var sp;
      if (tgt.classList && tgt.classList.contains("ud")) {
        /* 大型算符的上下限：^ 永远在上、_ 永远在下，来多顺序都对 */
        sp = mathNode(null);
        parseSeq(sp, s, r.a, r.b);
        if (which === "^") tgt.insertBefore(sp, tgt.firstChild);
        else tgt.appendChild(sp);
        if (tgt.childElementCount >= 2) pendingLim = null;
      } else {
        sp = mathNode(which === "^" ? "sup" : "sub");
        parseSeq(sp, s, r.a, r.b);
        tgt.appendChild(sp);
      }
      return r.next;
    }
    function parseCmd(out, s, i, b) {
      var j = i + 1, name = "";
      while (j < b && /[a-zA-Z]/.test(s[j])) name += s[j++];
      if (!name) {
        var c1 = s[j];
        if (c1 == null) return j;
        j++;
        if (c1 === "," || c1 === ";" || c1 === " ") { out.appendChild(mathNode("mo", "\u2009")); return j; }
        if (c1 === "!") return j;
        if (c1 === "\\") { out.appendChild(mathNode("mo", "\n")); return j; }
        name = c1;   /* \{ \| \% 这类转义符号 */
      }
      if (name === "left" || name === "right" || name === "big" ||
          name === "Big" || name === "bigg" || name === "Bigg" ||
          name === "bigl" || name === "bigr" || name === "displaystyle" ||
          name === "limits" || name === "nolimits") {
        if (name === "left" || name === "right") {
          /* 跳过紧跟的分隔符，按普通符号画 */
          if (s[j] === "\\") return parseCmd(out, s, j, b);
          if (s[j] === "." ) return j + 1;
          if (s[j] != null) out.appendChild(mathNode("mo", s[j]));
          return j + 1;
        }
        return j;
      }
      if (name === "quad") { out.appendChild(mathNode("mo", "\u2003")); return j; }
      if (name === "qquad") { out.appendChild(mathNode("mo", "\u2003\u2003")); return j; }

      if (name === "frac" || name === "dfrac" || name === "tfrac") {
        var r1 = readArg(s, j, b), r2 = readArg(s, r1.next, b);
        var f = mathNode("frac"), t1 = mathNode(null), t2 = mathNode(null);
        parseSeq(t1, s, r1.a, r1.b);
        parseSeq(t2, s, r2.a, r2.b);
        f.appendChild(t1); f.appendChild(t2);
        out.appendChild(f);
        return r2.next;
      }
      if (name === "sqrt") {
        var k2 = j;
        if (s[k2] === "[") { while (k2 < b && s[k2] !== "]") k2++; k2++; }
        var r = readArg(s, k2, b);
        var rt = mathNode("rt");
        rt.appendChild(mathNode("sym", "√"));
        var rad = mathNode("rad");
        parseSeq(rad, s, r.a, r.b);
        rt.appendChild(rad);
        out.appendChild(rt);
        return r.next;
      }
      if (name === "text" || name === "mathrm" || name === "textrm" ||
          name === "mbox" || name === "operatorname") {
        var rt2 = readArg(s, j, b);
        out.appendChild(mathNode("mtx", s.slice(rt2.a, rt2.b)));
        return rt2.next;
      }
      if (name === "mathbf" || name === "bm" || name === "boldsymbol") {
        var rt3 = readArg(s, j, b);
        var bw = mathNode("mnu");
        parseSeq(bw, s, rt3.a, rt3.b);
        bw.style.fontWeight = "700";
        out.appendChild(bw);
        return rt3.next;
      }
      if (name === "mathbb" || name === "Bbb") {
        var rt4 = readArg(s, j, b);
        var ch = s.slice(rt4.a, rt4.b);
        out.appendChild(mathNode("mnu", MBB[ch] || ch));
        return rt4.next;
      }
      if (MACC[name]) {
        var rt5 = readArg(s, j, b);
        var at = s.slice(rt5.a, rt5.b);
        out.appendChild(mathNode("mnu", at + MACC[name]));
        return rt5.next;
      }
      if (name === "lim") {
        var L2 = mathNode("lim");
        L2.appendChild(mathNode("bigo mnu", "lim"));
        var ud2 = mathNode("ud");
        L2.appendChild(ud2);
        out.appendChild(L2);
        pendingLim = ud2;
        return j;
      }
      if (BIGOPS[name]) {
        var sym = BIGOPS[name];
        if (/int/.test(name)) { out.appendChild(mathNode("bigo mo", sym)); return j; }
        var L = mathNode("lim");
        L.appendChild(mathNode("bigo", sym));
        var ud = mathNode("ud");
        L.appendChild(ud);
        out.appendChild(L);
        pendingLim = ud;
        return j;
      }
      if (name === "begin") {
        /* \begin{env}...\end{env}：cases 按行拆、矩阵按行列拆，
           其余环境降级成普通序列（别把整个公式吐成原文） */
        var er = readArg(s, j, b);
        var env = s.slice(er.a, er.b).trim();
        var endTag = "\\end{" + env + "}";
        var stop = s.indexOf(endTag, er.next);
        if (stop < 0) {
          out.appendChild(mathNode("mnu", s.slice(i, b)));
          return b;
        }
        var bodyS = er.next, bodyE = stop, next2 = stop + endTag.length;
        var delim = { pmatrix: ["(", ")"], pm: ["(", ")"],
                      bmatrix: ["[", "]"], Bmatrix: ["{", "}"],
                      vmatrix: ["‖", "‖"], Vmatrix: ["‖", "‖"],
                      matrix: ["", ""], cases: ["{", ""],
                      array: ["", ""], aligned: ["", ""], gathered: ["", ""] }[env];
        if (delim) {
          if (delim[0]) out.appendChild(mathNode("mo", delim[0]));
          var rows = splitRows(s, bodyS, bodyE);
          var tbl = mathNode("mendisp");
          for (var r = 0; r < rows.length; r++) {
            var rowEl = mathNode("mrow");
            var cols = splitCols(s, rows[r].a, rows[r].b);
            for (var c = 0; c < cols.length; c++) {
              if (c) rowEl.appendChild(mathNode("mo", "\u2003"));
              parseSeq(rowEl, s, cols[c].a, cols[c].b);
            }
            tbl.appendChild(rowEl);
          }
          out.appendChild(tbl);
          if (delim[1]) out.appendChild(mathNode("mo", delim[1]));
        } else {
          parseSeq(out, s, bodyS, bodyE);
        }
        return next2;
      }
      if (name === "end") {
        var er2 = readArg(s, j, b);
        return er2.next;   /* 多余的 \end 直接吞掉 */
      }
      if (GREEK[name]) { out.appendChild(mathNode("mi", GREEK[name])); return j; }
      if (MSYM[name]) { out.appendChild(mathNode("mo", MSYM[name])); return j; }
      if (MFUNCS[name]) { out.appendChild(mathNode("mnu", name)); out.appendChild(mathNode("mo", "\u2009")); return j; }
      /* 不认识的命令：原样吐出来，别吞字 */
      out.appendChild(mathNode("mnu", s.slice(i, j)));
      return j;
    }
    function parseSeq(out, s, a, b) {
      var i = a;
      var saveLim = pendingLim;
      pendingLim = null;
      while (i < b) {
        var ch = s[i];
        if (ch === "^" || ch === "_") i = parseSupSub(out, s, i, b, ch);
        else {
          /* 上下限只紧跟在大型算符后面，碰到普通记号就断开 */
          pendingLim = null;
          if (ch === "\\") i = parseCmd(out, s, i, b);
          else if (ch === "{") {
            var d = matchBrace(s, i, b);
            if (d < 0) { parseSeq(out, s, i + 1, b); i = b; }
            else { parseSeq(out, s, i + 1, d); i = d + 1; }
          }
          else if (ch === "}") i++;
          else if (/\s/.test(ch)) i++;
          else if ("+-=<>(),;.|!".indexOf(ch) >= 0) { out.appendChild(mathNode("mo", ch)); i++; }
          else if (/[0-9.]/.test(ch)) {
            var k3 = i;
            while (k3 < b && /[0-9.]/.test(s[k3])) k3++;
            out.appendChild(mathNode("mnu", s.slice(i, k3)));
            i = k3;
          }
          else { out.appendChild(mathNode("mi", ch)); i++; }
        }
      }
      pendingLim = saveLim;
    }
  }

  function renderMD(text, opts) {
    opts = opts || {};
    var base = opts.base || 0;
    var ranges = opts.ranges || null;
    var frag = document.createDocumentFragment();
    var lines = String(text == null ? "" : text).split("\n");
    var n = lines.length;
    var off = new Array(n + 1);
    off[0] = base;
    for (var q = 0; q < n; q++) off[q + 1] = off[q] + lines[q].length + 1;

    function push(node, i) {
      node.__end = (i >= n) ? (base + text.length) : off[i];
      frag.appendChild(node);
    }

    var i = 0;
    while (i < n) {
      var line = lines[i];

      /* 围栏代码块（流式时未闭合也算） */
      if (/^\s*```/.test(line)) {
        var lang = line.trim().slice(3).trim().split(/\s+/)[0] || "";
        i++;
        var buf = [];
        while (i < n && !/^\s*```/.test(lines[i])) buf.push(lines[i++]);
        if (i < n) i++;
        push(codeCard(lang, buf.join("\n"), opts.pending), i);
        continue;
      }

      if (!line.trim()) { i++; continue; }

      /* 块级公式 $$...$$（单行或多行） */
      if (/^\s*\$\$/.test(line)) {
        var one = /^\s*\$\$(.+)\$\$\s*$/.exec(line);
        if (one && one[1].trim()) {
          i++;
          push(mathRender(one[1].trim(), true), i);
          continue;
        }
        i++;
        var mbuf = [];
        while (i < n && !/^\s*\$\$/.test(lines[i])) mbuf.push(lines[i++]);
        if (i < n) i++;
        push(mathRender(mbuf.join("\n"), true), i);
        continue;
      }

      var h = /^\s{0,3}(#{1,6})\s+(.*)$/.exec(line);
      if (h) {
        var hd = document.createElement("h" + Math.min(h[1].length, 4));
        inline(hd, h[2].replace(/\s+#+\s*$/, ""), off[i] + (line.length - h[2].length), ranges);
        i++;
        push(hd, i);
        continue;
      }

      if (/^\s{0,3}([-*_])(\s*\1){2,}\s*$/.test(line)) {
        i++;
        push(document.createElement("hr"), i);
        continue;
      }

      /* 引用块：> 前缀会改变偏移，块内不吃标记 */
      if (/^\s{0,3}>/.test(line)) {
        var qb = [];
        while (i < n && /^\s{0,3}>/.test(lines[i])) {
          qb.push(lines[i].replace(/^\s{0,3}>\s?/, ""));
          i++;
        }
        var bq = document.createElement("blockquote");
        bq.appendChild(renderMD(qb.join("\n"), { base: 0, ranges: null }));
        push(bq, i);
        continue;
      }

      if (line.indexOf("|") !== -1 && i + 1 < n && /^\s*\|?\s*:?-{2,}/.test(lines[i + 1])) {
        var head = splitRow(line);
        i += 2;
        var rows = [];
        while (i < n && lines[i].indexOf("|") !== -1 && lines[i].trim()) rows.push(splitRow(lines[i++]));
        var wrap = document.createElement("div");
        wrap.className = "tablewrap";
        var tb = document.createElement("table");
        var thead = document.createElement("thead");
        var tr = document.createElement("tr");
        head.forEach(function (cell) {
          var th = document.createElement("th");
          inline(th, cell, 0, null);
          tr.appendChild(th);
        });
        thead.appendChild(tr);
        tb.appendChild(thead);
        var tbody = document.createElement("tbody");
        rows.forEach(function (r) {
          var rt = document.createElement("tr");
          for (var c = 0; c < head.length; c++) {
            var td = document.createElement("td");
            inline(td, r[c] == null ? "" : r[c], 0, null);
            rt.appendChild(td);
          }
          tbody.appendChild(rt);
        });
        tb.appendChild(tbody);
        wrap.appendChild(tb);
        push(wrap, i);
        continue;
      }

      var lm = /^(\s*)([-*+]|\d+[.)])\s+(.*)$/.exec(line);
      if (lm) {
        var ordered = /\d/.test(lm[2]);
        var lst = document.createElement(ordered ? "ol" : "ul");
        while (i < n) {
          var lm2 = /^(\s*)([-*+]|\d+[.)])\s+(.*)$/.exec(lines[i]);
          if (!lm2 || (/\d/.test(lm2[2]) !== ordered)) break;
          var li = document.createElement("li");
          var itemAbs = off[i] + lines[i].length - lm2[3].length;
          var task = /^\[( |x|X)\]\s+(.*)$/.exec(lm2[3]);
          if (task) {
            li.appendChild(document.createTextNode(task[1].toLowerCase() === "x" ? "☑ " : "☐ "));
            inline(li, task[2], itemAbs + (lm2[3].length - task[2].length), ranges);
          } else {
            inline(li, lm2[3], itemAbs, ranges);
          }
          lst.appendChild(li);
          i++;
        }
        push(lst, i);
        continue;
      }

      var pbuf = [line];
      var pStart = i;
      i++;
      while (i < n && lines[i].trim() && !isBlockStart(lines[i]) &&
             !/^\s{0,3}(#{1,6})\s/.test(lines[i])) {
        pbuf.push(lines[i++]);
      }
      var p = document.createElement("p");
      inline(p, pbuf.join("\n"), off[pStart], ranges);
      autolink(p);
      push(p, i);
    }

    return frag;
  }

  /* ═══ 4. 状态 ═══════════════════════════════════════════════════ */
  var nodes = [], fps = [], busy = false;
  var lastMsgs = [];        /* 当前会话快照，长按菜单要用 */
  var localMarks = {};      /* 本页自己打的标记（宿主不存，刷新即失效） */
  var STATUS_TXT = {
    awaitingConfirm: "待确认", running: "运行中", success: "完成",
    failed: "失败", rejected: "已停止"
  };
  var QCOLOR = {
    yellow: "#e4e800", green: "#72d6b5", blue: "#7fb6d9",
    pink: "#ef655d", gray: "#89938f"
  };

  function fmtMs(ms) {
    if (!ms || ms < 0) return "";
    return ms < 1000 ? (ms + "ms") : (Math.round(ms / 100) / 10).toFixed(1) + "s";
  }
  function toolSig(tools) {
    if (!tools || !tools.length) return "";
    var s = "";
    for (var k = 0; k < tools.length; k++) {
      var t = tools[k] || {};
      s += (t.name || "") + ":" + (t.status || "") + ":" +
           ((t.args || "") + "").length + ":" + ((t.output || "") + "").length + ";";
    }
    return s;
  }
  function fingerprint(m) {
    if (!m) return "";
    return [m.role, (m.content || "").length, (m.reasoning || "").length,
            m.pending ? 1 : 0, m.error ? 1 : 0, toolSig(m.tools),
            marksSig(m.marks), (m.quote && m.quote.text) || ""].join("|");
  }
  function el(tag, cls, txt) {
    var e = document.createElement(tag);
    if (cls) e.className = cls;
    if (txt != null) e.textContent = txt;
    return e;
  }
  function withLocal(i, m) {
    var arr = localMarks[i];
    if (!arr || !arr.length || !m) return m;
    var merged = {};
    for (var k in m) merged[k] = m[k];
    merged.marks = (m.marks || []).concat(arr);
    return merged;
  }
  function pretty(s) {
    s = String(s == null ? "" : s);
    try { return JSON.stringify(JSON.parse(s), null, 2); } catch (e) { return s; }
  }

  /* ═══ 5. 构建节点 ═══════════════════════════════════════════════ */
  function buildNode(m) {
    var row = el("div", "row");
    if (m.role === "user") {
      row.__quote = null;
      row.__me = el("div", "me");
      row.appendChild(row.__me);
      return row;
    }
    var ai = el("div", "ai");
    ai.__meta = el("div", "meta");
    ai.__tools = null;
    ai.__rel = null;
    ai.__relBox = null;
    ai.__toolBox = null;
    ai.__quote = null;
    ai.__body = el("div", "body");
    ai.__err = null;
    ai.appendChild(ai.__meta);
    ai.appendChild(ai.__body);
    row.appendChild(ai);
    return row;
  }

  /* 引用卡片：宿主的 message.quote = {text, color} */
  function paintQuote(host, parent, m, mine) {
    var q = m.quote;
    var hostKey = q && q.text ? (q.text + "|" + (q.color || "")) : "";
    if (!hostKey) {
      if (host.__quote) { host.__quote.remove(); host.__quote = null; }
      return;
    }
    if (!host.__quote) {
      host.__quote = el("div", "quote" + (mine ? " mine" : ""));
      host.__quoteQtx = el("div", "qt");
      host.__quote.appendChild(host.__quoteQtx);
      parent.insertBefore(host.__quote, parent.firstChild);
    }
    if (host.__quote.__key !== hostKey) {
      host.__quote.__key = hostKey;
      host.__quoteQtx.textContent = q.text;
      host.__quote.style.borderLeftColor = QCOLOR[q.color] || QCOLOR.yellow;
    }
  }

  /* 用户气泡：纯文本 + 标记 */
  /* 已发送消息的附件条：图片出缩略图（异步换 data URI），其余出芯片。
     点击走 meimg / mefile 两个 act。 */
  function attsSig(list) {
    if (!list || !list.length) return "";
    var s = "";
    for (var i = 0; i < list.length; i++) {
      s += (list[i] && (list[i].path || list[i].name) || "") + "~";
    }
    return s;
  }
  function buildMeAtts(list) {
    var wrap = el("div", "meatts");
    for (var i = 0; i < list.length; i++) {
      var a = list[i] || {};
      var name = a.name || String(a.path || "").split(/[\\/]/).pop() || "附件";
      var isImg = !!a.isImage || /\.(png|jpe?g|gif|webp|bmp|svg)$/i.test(name);
      if (isImg && a.path) {
        var im = document.createElement("img");
        im.className = "meimg loading";
        im.alt = name;
        im.setAttribute("data-act", "meimg");
        im.setAttribute("data-path", a.path);
        im.setAttribute("data-name", name);
        im.setAttribute("data-img", "1");
        im.addEventListener("load", function () { this.classList.remove("loading"); });
        im.addEventListener("error", function () {
          /* 解不出来就退化成文件名芯片，别留一个破图 */
          this.classList.remove("loading");
          this.style.display = "none";
        });
        wrap.appendChild(im);
        (function (img, path) {
          resolveImage(path, function (data, err) {
            if (data) { img.src = data; return; }
            /* 解不出来就退化成文件名芯片，别留一个灰方块 */
            img.classList.remove("loading");
            var fb = el("span", "mefile");
            fb.setAttribute("data-act", "mefile");
            fb.setAttribute("data-path", path);
            fb.setAttribute("data-name", img.alt || "图片");
            attIcon(fb, 16);
            fb.appendChild(el("b", null, img.alt || "图片"));
            if (img.parentNode) img.parentNode.replaceChild(fb, img);
            if (err) img.setAttribute("data-err", err);
          });
        })(im, a.path);
      } else {
        var f = el("span", "mefile");
        f.setAttribute("data-act", "mefile");
        f.setAttribute("data-path", a.path || "");
        f.setAttribute("data-name", name);
        attIcon(f, 16);
        f.appendChild(el("b", null, name));
        wrap.appendChild(f);
      }
    }
    return wrap;
  }

  function paintUser(row, m) {
    var me = row.__me, content = m.content || "";
    var atts = (m.attachments && m.attachments.length) ? m.attachments : null;
    var sig = content.length + "|" + marksSig(m.marks) + "|" + attsSig(atts);
    if (me.__sig === sig) return;
    me.__sig = sig;
    me.textContent = "";
    if (atts) me.appendChild(buildMeAtts(atts));
    if (content) emitText(me, content, 0, markRanges(content, m.marks));
    autolink(me);
  }

  /* 思考：有 reasoning 或 reasoningMs 就显示，流式期间默认展开 */
  function paintReasoning(ai, m) {
    var has = !!(m.reasoning && m.reasoning.length) || (m.reasoningMs > 0);
    if (!has) {
      if (ai.__rel) { ai.__rel.remove(); ai.__rel = null; }
      if (ai.__relBox) { ai.__relBox.remove(); ai.__relBox = null; }
      return;
    }
    if (!ai.__rel) {
      ai.__rel = el("button", "tag think");
      ai.__rel.type = "button";
      ai.__rel.setAttribute("data-act", "reason");
      ai.__relTxt = el("span");
      ai.__rel.appendChild(ai.__relTxt);
      ai.__rel.appendChild(el("i", "caret"));
      ai.__meta.insertBefore(ai.__rel, ai.__meta.firstChild);

      ai.__relBox = el("div", "fold reasoning");
      ai.__relBox.hidden = true;
      ai.__relBox.appendChild(el("div", "fttl", "思考过程"));
      ai.__relBox.appendChild(el("pre", null, ""));
      ai.insertBefore(ai.__relBox, ai.__body);
    }
    /* 结束之后不再写"思考中…"，也不留转圈 */
    var label = m.pending && !m.reasoning
      ? "思考中…"
      : (m.reasoningMs > 0 ? "思考 " + fmtMs(m.reasoningMs) : "思考过程");
    if (ai.__relTxt.textContent !== label) ai.__relTxt.textContent = label;

    var pre = ai.__relBox.lastChild;
    var rtxt = m.reasoning || "（模型没有返回思考内容）";
    if (pre.textContent !== rtxt) pre.textContent = rtxt;

    /* 默认一律折叠：生成中也不自动摊开（原来会自己弹开，很吵）。
       想看得自己点那枚"思考 · 2.4s"的签。 */
    ai.__rel.classList.toggle("open", !ai.__relBox.hidden);
  }

  function fillToolBox(ai, tt) {
    var key = (tt.name || "") + "|" + (tt.status || "") +
              "|" + ((tt.args || "") + "").length + "|" + ((tt.output || "") + "").length;
    if (ai.__toolBox.__key === key) return;
    ai.__toolBox.__key = key;
    var ttl = el("div", "fttl",
      (tt.name || "tool") + " · " + (STATUS_TXT[tt.status] || tt.status || ""));
    ai.__toolBox.textContent = "";
    ai.__toolBox.appendChild(ttl);
    if (tt.args) ai.__toolBox.appendChild(el("pre", null, pretty(tt.args)));
    if (tt.output) ai.__toolBox.appendChild(el("pre", null, String(tt.output)));
    if (!tt.args && !tt.output) ai.__toolBox.appendChild(el("pre", null, "还没有参数或输出。"));
  }

  /* 工具调用：默认只留一枚「工具 · 3」，点开才看到具体是哪几个。
     原来是直接把一串 chip 铺在正文上方，多点几次工具就把视线挡满了。 */
  function paintTools(ai, m) {
    var tools = m.tools || [];
    if (!tools.length) {
      if (ai.__toolSum) { ai.__toolSum.remove(); ai.__toolSum = null; }
      if (ai.__toolsBox) { ai.__toolsBox.remove(); ai.__toolsBox = null; }
      ai.__tools = null; ai.__toolBox = null; ai.__selTool = -1;
      return;
    }
    if (!ai.__toolSum) {
      ai.__toolSum = el("button", "tag toolsum");
      ai.__toolSum.type = "button";
      ai.__toolSum.setAttribute("data-act", "toolsum");
      ai.__toolSumTxt = el("span", "nm");
      ai.__toolSum.appendChild(ai.__toolSumTxt);
      ai.__toolSumN = el("span", "n");
      ai.__toolSum.appendChild(ai.__toolSumN);
      ai.__toolSum.appendChild(el("i", "caret"));
      ai.__meta.appendChild(ai.__toolSum);

      ai.__toolsBox = el("div", "fold toolsfold");
      ai.__toolsBox.hidden = true;          /* 默认折叠 */
      ai.__toolsBox.appendChild(el("div", "fttl", "工具调用"));
      ai.__tools = el("div", "tags");
      ai.__toolsBox.appendChild(ai.__tools);
      ai.insertBefore(ai.__toolsBox, ai.__body);
      ai.__toolsOpen = false;
    }

    if (ai.__tools.childElementCount !== tools.length) {
      ai.__tools.textContent = "";
      for (var k = 0; k < tools.length; k++) {
        var chip = el("button", "tag tool");
        chip.type = "button";
        chip.setAttribute("data-act", "tool");
        chip.setAttribute("data-i", String(k));
        chip.appendChild(el("span", "dot"));
        chip.appendChild(el("span", "nm"));
        chip.appendChild(el("span", "st"));
        ai.__tools.appendChild(chip);
      }
    }

    var failedN = 0, runningN = 0;
    for (var j = 0; j < tools.length; j++) {
      var t = tools[j] || {};
      var status = t.status || "running";
      if (status === "failed" || t.error) failedN++;
      if (status === "running" || status === "awaitingConfirm") runningN++;
      var chip2 = ai.__tools.children[j];
      if (chip2.__status !== status) {
        chip2.__status = status;
        chip2.className = "tag tool" + (status === "failed" || t.error ? " failed" : "");
        chip2.firstChild.className = "dot " + status;
      }
      var nm = t.name || "tool";
      if (chip2.children[1].textContent !== nm) chip2.children[1].textContent = nm;
      var stTxt = "· " + (STATUS_TXT[status] || status);
      if (chip2.children[2].textContent !== stTxt) chip2.children[2].textContent = stTxt;
      chip2.__tool = t;
      chip2.classList.toggle("active", ai.__selTool === j);
    }

    var sum = "工具 · " + tools.length;
    if (runningN) sum = "工具 · " + tools.length + " · 进行中";
    else if (failedN) sum = "工具 · " + tools.length + " · " + failedN + " 失败";
    if (ai.__toolSumTxt.textContent !== sum) ai.__toolSumTxt.textContent = sum;
    ai.__toolSumN.textContent = "";
    ai.__toolSum.classList.toggle("has-failed", failedN > 0);
    ai.__toolSum.classList.toggle("open", !ai.__toolsBox.hidden);

    if (ai.__selTool >= 0 && ai.__selTool < tools.length) {
      if (!ai.__toolBox) {
        ai.__toolBox = el("div", "fold toolfold");
        ai.__toolBox.appendChild(el("div", "fttl"));
        ai.__toolsBox.appendChild(ai.__toolBox);
      }
      fillToolBox(ai, tools[ai.__selTool] || {});
    } else if (ai.__toolBox) {
      ai.__toolBox.remove();
      ai.__toolBox = null;
    }
  }

  var PLAIN_LIMIT = 7000;

  /* 正文：流式时只重画"还没定稿"的那几块 */
  function paintBody(ai, m) {
    var body = ai.__body, content = m.content || "";
    var ranges = markRanges(content, m.marks);
    var plain = m.pending && content.length > PLAIN_LIMIT;

    if (plain) {
      if (!body.__plainMode) {
        body.textContent = "";
        body.appendChild(el("div", "plain"));
        body.__plainMode = true;
      }
      var pv = body.firstChild;
      if (pv.__src !== content) { pv.__src = content; pv.textContent = content; }
      body.__src = "";
      return;
    }

    if (body.__plainMode) {
      body.textContent = "";
      body.__plainMode = false;
      body.__src = "";
    }

    if (!content) {
      /* 空正文：只在还生成时给个呼吸点，结束就清干净 */
      if (m.pending) {
        if (!body.querySelector(".thinking")) {
          body.textContent = "";
          var dots = el("span", "thinking");
          dots.appendChild(el("i")); dots.appendChild(el("i")); dots.appendChild(el("i"));
          body.appendChild(dots);
        }
      } else if (body.firstChild) {
        body.textContent = "";
      }
      body.__src = "";
      return;
    }

    var prev = body.__src || "";
    var tail = body.querySelector(".caretline");
    if (tail) tail.remove();

    /* 流式 → 收尾：卡片要从"只有源码"升级成带预览/导图的完整形态，
       只重画尾巴不够，整条重画一次（只发生一次，代价可接受）。 */
    var flipped = body.__pending !== !!m.pending;
    body.__pending = !!m.pending;

    if (!flipped && prev && content.length > prev.length && content.indexOf(prev) === 0 &&
        body.childElementCount) {
      /* 追加式增长：保留已定稿的块，只重画尾巴 */
      var keep = null;
      var lastStableEnd = 0;
      for (var k = 0; k < body.childElementCount; k++) {
        var ch = body.children[k];
        var endOff = ch.__end;
        if (endOff == null || endOff >= prev.length) break;
        keep = k;
        lastStableEnd = endOff;
      }
      if (keep !== null) {
        while (body.childElementCount > keep + 1) body.removeChild(body.lastElementChild);
        body.appendChild(renderMD(content.slice(lastStableEnd), {
          base: lastStableEnd, ranges: ranges, pending: !!m.pending
        }));
        body.__src = content;
      } else {
        body.textContent = "";
        body.appendChild(renderMD(content, { base: 0, ranges: ranges, pending: !!m.pending }));
        body.__src = content;
      }
    } else if (flipped || body.__src !== content || body.__marks !== marksSig(m.marks)) {
      body.textContent = "";
      body.appendChild(renderMD(content, { base: 0, ranges: ranges, pending: !!m.pending }));
      body.__src = content;
    }
    body.__marks = marksSig(m.marks);

    if (m.pending && !body.querySelector(".caretline")) {
      body.appendChild(el("span", "caretline"));
    }
  }

  function paintNode(row, m) {
    if (m.role === "user") {
      paintQuote(row, row, m, true);
      if (!row.__me) row.appendChild(el("div", "me"));
      paintUser(row, m);
      return;
    }
    var ai = row.firstChild;
    ai.classList.toggle("pending", !!m.pending);
    ai.classList.toggle("idle", !busy && !m.pending);
    paintReasoning(ai, m);
    paintTools(ai, m);
    paintQuote(ai, ai, m, false);
    paintBody(ai, m);

    if (m.error) {
      if (!ai.__err) { ai.__err = el("div", "err"); ai.appendChild(ai.__err); }
      if (ai.__err.textContent !== m.error) ai.__err.textContent = m.error;
    } else if (ai.__err) {
      ai.__err.remove(); ai.__err = null;
    }
  }

  /* ═══ 6. 滚动与回到底部 ═════════════════════════════════════════ */
  var jumpOn = false, jumpDirty = false;
  function nearBottom() {
    return listEl.scrollHeight - listEl.scrollTop - listEl.clientHeight < 90;
  }
  function syncJumpNow() {
    var atBottom = nearBottom();
    var on = !atBottom, dirty = on && busy;
    if (on !== jumpOn) { jumpEl.classList.toggle("on", on); jumpOn = on; }
    if (dirty !== jumpDirty) { jumpEl.classList.toggle("dirty", dirty); jumpDirty = dirty; }
    updateThumb();
  }
  /* scroll 事件在某些 WebView 里一帧能来四五次，每次都读 scrollHeight /
     scrollTop 会逼出同步布局 —— 合并到一帧里做一次就够。 */
  var jumpRaf = 0;
  function syncJump() {
    if (jumpRaf) return;
    jumpRaf = requestAnimationFrame(function () { jumpRaf = 0; syncJumpNow(); });
  }
  listEl.addEventListener("scroll", syncJump, { passive: true });

  /* 滚动期间摘掉输入条的 backdrop-filter（见 CSS 注释），停手 180ms 再还回去 */
  var scrollIdle = 0;
  listEl.addEventListener("scroll", function () {
    var b = document.body;
    if (!b.classList.contains("scrolling")) b.classList.add("scrolling");
    clearTimeout(scrollIdle);
    scrollIdle = setTimeout(function () { b.classList.remove("scrolling"); }, 180);
  }, { passive: true });

  /* 跳到底 + 拖拽：按住上下拖，过阈值松手跳到顶/底 */
  function jumpToBottom() { listEl.scrollTop = listEl.scrollHeight; syncJump(); }
  jumpEl.addEventListener("click", function () { jumpToBottom(); });

  (function () {
    var startY = 0, dy = 0, dragging = false, suppressClick = false;
    var TH = 36;
    jumpEl.addEventListener("pointerdown", function (e) {
      if (e.button && e.button !== 0) return;
      dragging = true; suppressClick = false;
      startY = e.clientY; dy = 0;
      try { jumpEl.setPointerCapture(e.pointerId); } catch (err) {}
    });
    jumpEl.addEventListener("pointermove", function (e) {
      if (!dragging) return;
      dy = e.clientY - startY;
      if (Math.abs(dy) > 6) suppressClick = true;
      jumpEl.classList.toggle("dragging", Math.abs(dy) > 6);
      if (dy < -TH) {
        jhintEl.textContent = "到顶 ↑";
        jumpEl.style.transform = "translateY(" + Math.max(dy, -TH * 1.6) + "px)";
      } else if (dy > TH) {
        jhintEl.textContent = "到底 ↓";
        jumpEl.style.transform = "translateY(" + Math.min(dy, TH * 1.6) + "px)";
      } else {
        jhintEl.textContent = Math.abs(dy) > 6 ? "上下拖动" : "";
        jumpEl.style.transform = dy ? "translateY(" + (dy * 0.4) + "px)" : "";
      }
    });
    function end() {
      if (!dragging) return;
      dragging = false;
      jumpEl.classList.remove("dragging");
      jumpEl.style.transform = "";
      if (dy < -TH) {
        listEl.scrollTop = 0; syncJump();     /* 上拖 → 到顶 */
      } else if (dy > TH) {
        jumpToBottom();                        /* 下拖 → 到底 */
      }
      dy = 0;
    }
    jumpEl.addEventListener("pointerup", end);
    jumpEl.addEventListener("pointercancel", end);
    /* 拖过的松手不再触发 click 的"回到底"重复动作 */
    jumpEl.addEventListener("click", function (e) {
      if (suppressClick) { suppressClick = false; e.stopPropagation(); e.preventDefault(); }
    }, true);
  })();

  /* ── 键盘 / 视口：动画期间一律不做重活 ───────────────────────────
     以前的写法是每次 visualViewport resize 就把 body 高度改成 px：
     键盘升降是 200~300ms 的连续动画，那等于**每帧**重新布局整棵 flex 列、
     重新分配滚动容器的高度 —— 两个后果：
       1. 动画本身掉帧（每帧一次全页重排 + 合成层重建）；
       2. 容器长回去的那一段从来没画过 —— 收键盘后再往下滑，
          底部先空一块，滑一下才"闪"出来。
     改成三段：
       · 动画进行中：只挂 body.kb（关玻璃模糊），不量、不写布局；
       · 停手 170ms：才改高度、量输入框、垫底、补画空白瓦片；
       · 原生已经自己缩了布局视口（adjustResize）就**完全不写**高度，
         让 CSS 的 height:100% 跟着走 —— 浏览器处理 resize 比 JS
         每帧写一遍省得多。 */
  var composerEl = document.querySelector(".composer");
  function syncComposerPad() {
    if (!composerEl || !composerEl.offsetHeight) return;
    var h = composerEl.offsetHeight;
    /* 附件条 / 待发引用条会把输入区临时撑高，只把这一截垫进列表底部，
       最后一条消息就不会被压到框底下；平时留 20px 呼吸。 */
    var extra = (qbarEl ? qbarEl.offsetHeight : 0) +
                (attachBar ? attachBar.offsetHeight : 0);
    var pad = (20 + extra) + "px";
    if (listEl.style.paddingBottom !== pad) {
      listEl.style.paddingBottom = pad;
      /* 底部柔光贴在输入框上沿 —— 用 CSS 变量喂给 .edgeglow.bottom */
      document.documentElement.style.setProperty("--composer-h", h + "px");
    }
  }

  /* 贴底状态跟着滚动实时更新；输入区高度一变，贴底的就重新贴一次。
     写滚动位置合并到下一帧，别和输入框量高撞在同一帧里反复逼同步布局。 */
  var stickBottom = true, stickRaf = 0;
  listEl.addEventListener("scroll", function () { stickBottom = nearBottom(); }, { passive: true });
  if (window.ResizeObserver && composerEl) {
    try {
      new ResizeObserver(function () {
        if (!stickBottom || stickRaf) return;
        stickRaf = 1;
        requestAnimationFrame(function () {
          stickRaf = 0;
          var gap = listEl.scrollHeight - listEl.scrollTop - listEl.clientHeight;
          if (gap > 1) listEl.scrollTop = listEl.scrollHeight;   /* 已经在底部就别写 */
        });
      }).observe(composerEl);
    } catch (e) {}
  }

  function layoutH() {
    return document.documentElement.clientHeight || window.innerHeight || 0;
  }

  /* 只有"布局视口没跟着缩"的 WebView（adjustPan / 高度不同步）才需要
     手动把 body 压到可视高度；adjustResize 下 clientHeight 已经跟着变了，
     再写一次就是多一次 resize。 */
  function followViewport() {
    var vv = window.visualViewport;
    if (!vv) { syncComposerPad(); return; }
    var vh = Math.round(vv.height), lh = layoutH();
    if (vh > 160 && lh - vh > 2) {
      var h = vh + "px";
      if (document.body.style.height !== h) document.body.style.height = h;
    } else if (document.body.style.height) {
      document.body.style.height = "";      /* 回到 100%，交给 CSS */
    }
  }

  /* 容器尺寸变过之后，WebView 的滚动层常留着一块"没画过"的瓦片：
     滑到那里先空白、再闪回内容。这里在**同一帧内**把列摘下来再挂回去，
     逼渲染器丢掉旧瓦片重新排一次（同步完成，用户看不到中间态）。
     只做一次，代价是一帧的重排，换来的是"不再有空白带"。 */
  var healTries = 0;
  function healPaint() {
    if (!listEl.childElementCount) return;
    /* 正在惯性滑动时别动 —— 摘一次列会把惯性打断。等它停手
       （body.scrolling 撤掉）再补画；连着滑就一直往后排。 */
    if (document.body.classList.contains("scrolling") && healTries < 6) {
      healTries++;
      setTimeout(healPaint, 240);
      return;
    }
    healTries = 0;
    var st = listEl.scrollTop;
    listEl.style.display = "none";
    void listEl.offsetHeight;        /* 强制同步布局：旧布局对象在这里被丢掉 */
    listEl.style.display = "";
    listEl.scrollTop = st;
  }

  var kbSettle = 0, kbStick = true, kbOn = false;
  function onViewport() {
    /* 只在"这一轮"开始时取一次是否贴底：动画期间高度一直在变，
       每帧判断会得到不一致的结果。 */
    if (!kbOn) { kbOn = true; kbStick = nearBottom(); }
    document.body.classList.add("kb");
    clearTimeout(kbSettle);
    kbSettle = setTimeout(settleViewport, 170);
  }
  function settleViewport() {
    kbOn = false;
    document.body.classList.remove("kb");
    followViewport();
    autosize();
    syncComposerPad();
    healPaint();
    if (kbStick) listEl.scrollTop = listEl.scrollHeight;
    syncJump();
    scheduleScrub();
  }

  if (window.visualViewport) {
    window.visualViewport.addEventListener("resize", onViewport);
  }
  /* 有的 WebView 只给 window resize、visualViewport 不给事件 ——
     两条都挂到同一个处理上，谁先来都算数（内部按 170ms 合并）。 */
  window.addEventListener("resize", onViewport);
  followViewport();
  syncComposerPad();

  /* ── 附件条点击：查看图片 / 删除 ──────────────────────────────── */
  attachBar.addEventListener("click", function (e) {
    var t = e.target;
    if (!t || !t.closest) return;
    var chip = t.closest(".att");
    if (!chip) return;
    var idx = chip.dataset.i | 0;
    var act = t.closest("[data-act]");
    var kind = act ? act.getAttribute("data-act") : "";

    if (kind === "delatt") {
      /* 先乐观删掉，宿主的 onState 随后会把权威状态推回来 */
      var remain = (lastAttList || []).filter(function (_, k) { return k !== idx; });
      lastAttList = remain;
      chip.remove();
      if (!attsEl.childElementCount) attachBar.classList.remove("on");
      syncComposerPad();
      if (HAS_BRIDGE) call("removeAttachment", { index: idx });
      else toast("预览模式没有宿主，删不掉真附件");
      return;
    }
    if (kind === "viewatt" || chip.getAttribute("data-img") === "1") {
      var a = (lastAttList || [])[idx] || {};
      /* 非图片附件：一律走"本地读取"预览 —— 文本摊开看内容，
         二进制/读不到会在预览层里给提示和「打开」出口。
         附件是用户自己选进来给模型的，理应能点开看。 */
      if (chip.getAttribute("data-img") !== "1" && !a.isImage) {
        if (!a.path) { toast("这个附件没有路径，读不了"); return; }
        previewFile(a.path, a.name);
        return;
      }
      var im = chip.querySelector("img");
      var shown = im && im.src && im.src.indexOf("data:") === 0 ? im.src : "";
      if (shown) { zoomImg(shown, a.name || "附件图片"); return; }
      /* 缩略图还没解出来 / 解失败：现场再要一次 data URI，拿不到提示 */
      toast("图片读取中…");
      resolveImage(a.path, function (data, err) {
        if (data) zoomImg(data, a.name || "附件图片");
        else toast(err || "这张图读不到");
      });
    }
  });

  /* ═══ 6.4 到顶/到底：只给一层边缘柔光 ────────────────────────────
     上一版是自己做的橡皮筋（位移 + 欠阻尼弹簧回位）。那个"弹回来"
     既不像 iOS 的橡皮筋，又比原生硬停更生硬 —— 整段删掉。

     现在的反馈是"看见"而不是"位移"：内容一动不动，到边还继续往外拉
     的时候，边缘浮起一层很淡的光，松手 200ms 化开。
     实现上只写 opacity（合成层属性）：不触发布局、不改滚动位置、
     不重建瓦片 —— 是这一页里能想到最省的一种"到头感"。 */
  (function () {
    var MAXO = 0.18;      /* 最亮时的透明度：看得见，又不脏 */
    var FULL = 88;        /* 往外拉到 88px 就满，之后不再加亮 */

    function mk(cls) {
      var d = document.createElement("div");
      d.className = "edgeglow " + cls;
      return d;
    }
    var topEl = mk("top"), botEl = mk("bottom");
    var stage = listEl.parentNode;
    stage.insertBefore(topEl, listEl);        /* 垫在列表底下，正文还压得住 */
    stage.appendChild(botEl);

    var tracking = false, active = "", over = 0, lastY = 0, rafId = 0;

    function show(el, k) {
      el.classList.add("live");
      el.style.opacity = (k * MAXO).toFixed(3);
    }
    function hide(el) {
      el.classList.remove("live");
      el.style.opacity = "";
    }
    function flush() {
      rafId = 0;
      var k = Math.min(1, over / FULL);
      if (active === "top") { show(topEl, k); hide(botEl); }
      else if (active === "bottom") { show(botEl, k); hide(topEl); }
      else { hide(topEl); hide(botEl); }
    }
    function schedule() {
      if (rafId) return;
      rafId = requestAnimationFrame(flush);
    }
    function maxTop() {
      return Math.max(0, listEl.scrollHeight - listEl.clientHeight);
    }

    listEl.addEventListener("touchstart", function (e) {
      if (e.touches.length !== 1 || scrubbing) { tracking = false; return; }
      tracking = true; active = ""; over = 0;
      lastY = e.touches[0].clientY;
    }, { passive: true });

    /* 只读数据、不 preventDefault：滚动 / 惯性 / 右缘定位一律不受影响。
       over 按"逐帧增量"累加，往回推会减回去，所以拉出去再推回来光会跟着退。 */
    listEl.addEventListener("touchmove", function (e) {
      if (!tracking || e.touches.length !== 1) return;
      var y = e.touches[0].clientY, d = y - lastY;
      lastY = y;
      if (!d) return;

      var st = listEl.scrollTop;
      if (active === "top") {
        over += d;
        if (over <= 0) { over = 0; active = ""; }
      } else if (active === "bottom") {
        over -= d;
        if (over <= 0) { over = 0; active = ""; }
      } else if (d > 0 && st <= 0.5) {
        active = "top"; over = d;
      } else if (d < 0 && maxTop() > 0 && st >= maxTop() - 0.5) {
        active = "bottom"; over = -d;
      }
      schedule();
    }, { passive: true });

    function end() {
      if (!tracking) return;
      tracking = false; active = ""; over = 0;
      schedule();
    }
    listEl.addEventListener("touchend", end);
    listEl.addEventListener("touchcancel", end);
    /* 惯性 / 程序滚动不算"手指接触"，别没事闪一下 */
    listEl.addEventListener("scroll", function () {
      if (!tracking && over === 0) { hide(topEl); hide(botEl); }
    }, { passive: true });
  })();

  /* ═══ 6.5 右缘滑动定位器：右滑呼出，拖动定位到用户气泡 ═════════ */
  var scrubEl = $("scrub"), srail = $("srail"), sthumb = $("sthumb"),
      sticksEl = $("sticks"), scardEl = $("scard"), scardTxt = $("scardTxt");
  var scrubbing = false, railTop = 0, railH = 0, thumbH = 34, hideTimer = 0;
  var tickRows = [], tickCount = -1;

  function maxScroll() { return Math.max(0, listEl.scrollHeight - listEl.clientHeight); }
  function canScrub() { return maxScroll() > 24; }
  function showScrub() { scrubEl.classList.add("on"); clearTimeout(hideTimer); }
  function scheduleHide() {
    clearTimeout(hideTimer);
    hideTimer = setTimeout(function () {
      if (!scrubbing) scrubEl.classList.remove("on");
    }, 1600);
  }
  function rebuildTicks() {
    if (nodes.length === tickCount) return;
    tickCount = nodes.length;
    sticksEl.textContent = "";
    tickRows = [];
    for (var i = 0; i < nodes.length; i++) {
      var m = lastMsgs[i];
      if (!m || m.role !== "user") continue;
      var s = el("i", "stick");
      sticksEl.appendChild(s);
      tickRows.push({ el: s, row: nodes[i], idx: i, y: 0 });
    }
  }
  function updateThumb() {
    var ms = maxScroll();
    if (!ms || !railH) return;
    var y = listEl.scrollTop / ms * (railH - thumbH);
    sthumb.style.top = Math.max(0, Math.min(railH - thumbH, y)) + "px";
    if (!scrubbing) return;
    var mid = y + thumbH / 2, best = -1, bd = 1e9;
    for (var i = 0; i < tickRows.length; i++) {
      var d = Math.abs(tickRows[i].y - mid);
      if (d < bd) { bd = d; best = i; }
    }
    for (var j = 0; j < tickRows.length; j++) {
      tickRows[j].el.classList.toggle("cur", j === best);
    }
  }
  function layoutScrub() {
    if (!canScrub()) {
      if (!scrubbing) scrubEl.classList.remove("on");
      return;
    }
    var r = srail.getBoundingClientRect();
    railTop = r.top; railH = r.height;
    thumbH = Math.max(32, railH * listEl.clientHeight / listEl.scrollHeight);
    sthumb.style.height = thumbH + "px";
    var sh = listEl.scrollHeight;
    for (var i = 0; i < tickRows.length; i++) {
      var t = tickRows[i];
      t.y = Math.min(railH - 2, Math.max(0, t.row.offsetTop / sh * railH));
      t.el.style.top = t.y + "px";
    }
    updateThumb();
  }
  /* layoutScrub 里有 getBoundingClientRect + 一串 offsetTop，全是强制布局。
     流式时它被 flush / onPatch 每帧调一次，等于每帧多算一遍整页几何 ——
     合并到下一帧做一次。 */
  var scrubRaf = 0;
  function scheduleScrub() {
    if (scrubRaf) return;
    scrubRaf = requestAnimationFrame(function () {
      scrubRaf = 0;
      layoutScrub();
    });
  }
  function seekTo(y) {
    var ms = maxScroll();
    var ratio = (y - railTop - thumbH / 2) / Math.max(1, railH - thumbH);
    ratio = Math.max(0, Math.min(1, ratio));
    var target = ratio * ms;
    var best = null, bd = 1e9;
    for (var i = 0; i < tickRows.length; i++) {
      var st = Math.max(0, Math.min(ms, tickRows[i].row.offsetTop - 12));
      var d = Math.abs(st - target);
      if (d < bd) { bd = d; best = { st: st, idx: tickRows[i].idx, t: tickRows[i] }; }
    }
    if (best && bd < Math.max(40, ms * 0.045)) {
      target = best.st;
      scardTxt.textContent = (lastMsgs[best.idx].content || "").trim()
        .slice(0, 44) || "（空）";
      scardEl.style.top = (best.t.y + 12) + "px";
    }
    listEl.scrollTop = target;
  }
  function startDrag(y) {
    if (!canScrub()) return false;
    showScrub();
    scrubbing = true;
    scrubEl.classList.add("dragging");
    layoutScrub();
    seekTo(y);
    return true;
  }
  function moveDrag(y) { if (scrubbing) seekTo(y); }
  function endDrag() {
    if (!scrubbing) return;
    scrubbing = false;
    scrubEl.classList.remove("dragging");
    for (var j = 0; j < tickRows.length; j++) tickRows[j].el.classList.remove("cur");
    updateThumb();
    scheduleHide();
  }

  /* 右缘 34px 内按下 = 直接进入定位模式 */
  listEl.addEventListener("touchstart", function (e) {
    if (e.touches.length !== 1) return;
    if (e.touches[0].clientX < window.innerWidth - 34) return;
    clearTimeout(pressTimer);
    if (startDrag(e.touches[0].clientY)) e.preventDefault();
  }, { passive: false });
  listEl.addEventListener("touchmove", function (e) {
    if (!scrubbing || e.touches.length !== 1) return;
    e.preventDefault();
    moveDrag(e.touches[0].clientY);
  }, { passive: false });
  listEl.addEventListener("touchend", endDrag);
  listEl.addEventListener("touchcancel", endDrag);

  /* 桌面预览：拖滑轨 */
  srail.addEventListener("mousedown", function (e) {
    if (!startDrag(e.clientY)) return;
    e.preventDefault();
    var mv = function (ev) { moveDrag(ev.clientY); };
    var up = function () {
      document.removeEventListener("mousemove", mv);
      document.removeEventListener("mouseup", up);
      endDrag();
    };
    document.addEventListener("mousemove", mv);
    document.addEventListener("mouseup", up);
  });

  /* ═══ 7. onMessages（rAF 合并，避免推送密集时重复排版）══════════ */
  var pendingMsgs = null, pendingBusy = false, frame = 0;

  /* 宿主偶发：busy 已经 false，最后一条消息的 pending 还是 true。
     表现就是按钮从「暂停」变回「发送」了，消息尾巴却还挂着转圈 / 光标，
     卡片也一直停在"生成中"。以 busy 为准 —— 既然不在生成，
     就不该有任何一条还在 pending。 */
  function settlePending(msgs, b) {
    if (b || !msgs || !msgs.length) return msgs;
    var last = msgs[msgs.length - 1];
    if (!last || !last.pending) return msgs;
    var c = {};
    for (var k in last) c[k] = last[k];
    c.pending = false;
    var out = msgs.slice();
    out[out.length - 1] = c;
    return out;
  }

  function flush() {
    frame = 0;
    var msgs = pendingMsgs || [];
    pendingMsgs = null;
    var b = pendingBusy;
    msgs = settlePending(msgs, b);

    /* 快照刚恢复完，宿主第一条全量紧跟而来 —— 如果两边是**同一批消息**，
       就别重画了：重建节点会让卡片/iframe 闪一下，正好抵消了恢复的意义。
       只把数据源换成宿主给的完整版（快照里正文/工具输出是截断的）。 */
    var adopt = false;
    if (restoredTick && nodes.length && msgs.length === nodes.length) {
      restoredTick = false;
      adopt = true;
      for (var q = 0; q < msgs.length; q++) {
        var nm = msgs[q] || {}, om = lastMsgs[q] || {};
        if (nm.role !== om.role) { adopt = false; break; }
        if (String(nm.content || "").length !== (om.clen != null ? om.clen : String(om.content || "").length)) {
          adopt = false; break;
        }
        if (q === 0 && String(nm.content || "").slice(0, 24) !== String(om.content || "").slice(0, 24)) {
          adopt = false; break;
        }
      }
    } else {
      restoredTick = false;
    }

    lastMsgs = msgs;
    var stick = nearBottom();

    if (!msgs.length) {
      if (nodes.length) {
        for (var d = 0; d < nodes.length; d++) nodes[d].remove();
        nodes.length = 0; fps.length = 0;
      }
      if (!listEl.contains(emptyEl)) { listEl.textContent = ""; listEl.appendChild(emptyEl); }
      /* 宿主推来空列表 = 这是一个新聊天，旧快照必须扔掉，
         否则下次进来又会先闪一下上一个会话的内容。 */
      snapClear();
      syncJump();
      return;
    }

    if (emptyEl.parentNode) emptyEl.remove();

    if (msgs.length < nodes.length) {
      for (var r = 0; r < nodes.length; r++) nodes[r].remove();
      nodes.length = 0; fps.length = 0;
    }

    for (var i = 0; i < msgs.length; i++) {
      var m = withLocal(i, msgs[i] || {});
      var fp = fingerprint(m) + (i === msgs.length - 1 ? "|" + (b ? 1 : 0) : "");
      /* 单条渲染失败只降级这一条，别把整页拖死 */
      try {
        if (adopt && i < nodes.length) {
          /* 内容跟快照一模一样：沿用旧节点，只把指纹换成宿主这份的，
             后续有增量再按指纹重画。 */
          fps[i] = fp;
        } else if (i >= nodes.length) {
          var row = buildNode(m);
          listEl.appendChild(row);
          nodes.push(row);
          fps.push("");
          paintNode(row, m);
          fps[i] = fp;
        } else if (fps[i] !== fp) {
          paintNode(nodes[i], m);
          fps[i] = fp;
        }
      } catch (err) {
        if (i >= nodes.length) {
          /* 新建就失败：降级成纯文本气泡，节点数保持对齐 */
          var fb = el("div", "row");
          var me2 = el("div", "me");
          me2.textContent = String(m.content || m.text || "（这条消息渲染失败）");
          fb.appendChild(me2);
          listEl.appendChild(fb);
          nodes.push(fb);
          fps.push(fp);
        } else {
          /* 已有节点重绘失败：保留旧样子，只更新指纹 */
          fps[i] = fp;
        }
      }
    }
    while (nodes.length > msgs.length) {
      var tail = nodes.pop();
      fps.pop();
      tail.remove();
    }

    if (stick) listEl.scrollTop = listEl.scrollHeight;
    syncJump();
    rebuildTicks();
    scheduleScrub();
    snapSchedule(msgs);
  }

  window.App.onMessages = function (msgs, b) {
    /* 容错：宿主万一推了非数组（null / 类数组 / 单对象）也不至于崩 */
    if (!Array.isArray(msgs)) {
      msgs = (msgs && typeof msgs.length === "number")
        ? Array.prototype.slice.call(msgs) : [];
    }
    busy = !!b;
    setBusyUI(busy);
    pendingMsgs = msgs;
    pendingBusy = busy;
    if (!frame) frame = raf(flush);
  };

  /* 单条增量：宿主在"条数没变"时只推最后一条。
     不实现也能用（宿主会退回全量），但流式时每 120ms 重排整屏
     几十条消息，正是"打字发卡"的主因 —— 这里只重画变化的那一行。 */
  window.App.onPatch = function (i, msg, b) {
    if (typeof i !== "number" || i < 0 || !msg) return;
    if (!Array.isArray(lastMsgs)) lastMsgs = [];
    while (lastMsgs.length < i) lastMsgs.push({ role: "assistant", content: "" });
    /* 同 flush：busy=false 时最后一条不该还挂着 pending */
    if (!b && i === lastMsgs.length - 1 && msg.pending) {
      var c0 = {};
      for (var k0 in msg) c0[k0] = msg[k0];
      c0.pending = false;
      msg = c0;
    }
    lastMsgs[i] = msg;

    busy = !!b;
    setBusyUI(busy);

    if (!nodes[i]) {
      /* 节点还没建（刚好在列表末尾新增）：走一次全量更省心 */
      pendingMsgs = lastMsgs.slice();
      pendingBusy = busy;
      if (!frame) frame = raf(flush);
      return;
    }
    var stick = nearBottom();
    var m = withLocal(i, msg);
    var fp = fingerprint(m) + (i === lastMsgs.length - 1 ? "|" + (b ? 1 : 0) : "");
    if (fps[i] !== fp) {
      try { paintNode(nodes[i], m); } catch (e) {}
      fps[i] = fp;
    }
    if (stick) listEl.scrollTop = listEl.scrollHeight;
    syncJump();
    scheduleScrub();
    snapSchedule(lastMsgs);
  };

  /* ═══ 7.5 会话快照：重载后立刻接上"这是哪个聊天" ══════════════
     宿主 onLoadStop 后**确实**会推一遍全量，但那要等页面就绪 + 跨桥 +
     JSON 解析；这几百毫秒里列表是空的，看起来就是"刷新一下聊天没了"。
     所以每次收到消息都留一份轻量快照，页面启动时先画出来，
     宿主的全量一到就覆盖（同一会话内容一致，用户看不出换过手）。

     只存骨架：正文截断、思考清空、工具不留输出 ——
     localStorage 只有 5MB，几十条长会话原样塞进去会写失败。 */
  var SNAP_KEY = "mojian.session.v3";
  var SNAP_TTL = 36 * 3600 * 1000;   /* 超过一天半就不认了，免得张冠李戴 */
  var snapTimer = 0;
  var restoredTick = false;   /* 刚用快照画完，等宿主第一份完整数据"接管" */

  function snapDigest(msgs) {
    var head = "";
    for (var i = 0; i < msgs.length && i < 6; i++) {
      var m = msgs[i];
      if (m && m.role === "user" && m.content) { head = String(m.content).slice(0, 48); break; }
    }
    var tail = msgs.length ? String((msgs[msgs.length - 1] || {}).content || "") : "";
    return msgs.length + "|" + head + "|" + tail.length;
  }
  function snapShrink(msgs) {
    var from = Math.max(0, msgs.length - 40);   /* 只留最近 40 条 */
    var out = [];
    for (var i = from; i < msgs.length; i++) {
      var m = msgs[i] || {};
      var tools = [];
      var ts = m.tools || [];
      for (var k = 0; k < ts.length && k < 8; k++) {
        tools.push({ name: ts[k].name, summary: ts[k].summary || ts[k].name,
                     status: ts[k].status, args: "", output: "",
                     error: !!ts[k].error });
      }
      var full = String(m.content || "");
      out.push({
        role: m.role,
        clen: full.length,                 /* 原文长度：adopt 判定用 */
        content: full.slice(0, 12000),
        reasoning: "", reasoningMs: 0,
        pending: false, error: m.error || null,
        tools: tools,
        marks: (m.marks || []).slice(0, 20), quote: m.quote || null
      });
    }
    return out;
  }
  /* 会话快照写入。localStorage 是**同步 IO**，几十 KB 的 JSON 序列化
     + 写盘会跟流式渲染抢主线程（一写就掉几帧）。所以：
       · 流式期间延迟拉长（1.6s），收尾后再写；
       · 序列化放到 idle 里做，实在空闲不下来也会在 1.2s 后兜底；
       · digest 没变就不写 —— 聊天没变化时一个字节都不碰盘。 */
  var lastSnapDigest = "";
  function snapSchedule(msgs) {
    if (!HAS_BRIDGE || !msgs || !msgs.length) return;
    if (snapTimer) clearTimeout(snapTimer);
    snapTimer = setTimeout(function () {
      snapTimer = 0;
      if (!lastMsgs || !lastMsgs.length) return;
      var dg = snapDigest(lastMsgs);
      if (dg === lastSnapDigest) return;
      var write = function () {
        try {
          localStorage.setItem(SNAP_KEY, JSON.stringify({
            t: Date.now(),
            digest: dg,
            bottom: Math.max(0, listEl.scrollHeight - listEl.scrollTop - listEl.clientHeight),
            msgs: snapShrink(lastMsgs)
          }));
          lastSnapDigest = dg;
        } catch (e) {}
      };
      if (window.requestIdleCallback) {
        requestIdleCallback(write, { timeout: 1200 });
      } else {
        setTimeout(write, 0);
      }
    }, busy ? 1600 : 600);
  }
  function snapLoad() {
    try {
      var raw = localStorage.getItem(SNAP_KEY);
      if (!raw) return null;
      var o = JSON.parse(raw);
      if (!o || !Array.isArray(o.msgs) || !o.msgs.length) return null;
      if (Date.now() - (o.t || 0) > SNAP_TTL) { localStorage.removeItem(SNAP_KEY); return null; }
      return o;
    } catch (e) { return null; }
  }
  function snapClear() {
    try { localStorage.removeItem(SNAP_KEY); } catch (e) {}
  }
  /* 启动时还原。宿主随后推来的全量会走 flush 覆盖掉；若推来的是空
     数组（用户新开了一个聊天），也会走 flush 把列表清干净。 */
  function restoreSnapshot() {
    if (!HAS_BRIDGE) return false;         /* 桌面预览不掺和 */
    var s = snapLoad();
    if (!s) return false;
    lastMsgs = s.msgs;
    pendingMsgs = s.msgs;
    pendingBusy = false;
    flush();                                /* 同步画，别等一帧，越快越不闪 */
    restoredTick = true;
    var gap = s.bottom | 0;
    if (gap > 2) {
      listEl.scrollTop = Math.max(0, listEl.scrollHeight - listEl.clientHeight - gap);
    } else {
      listEl.scrollTop = listEl.scrollHeight;
    }
    syncJump();
    return true;
  }

  /* ═══ 8. onState ════════════════════════════════════════════════ */
  var lastModel = "", lastAtt = -1;
  window.App.onState = function (s) {
    if (!s) return;
    if (typeof s.busy === "boolean" && s.busy !== busy) {
      busy = s.busy;
      setBusyUI(busy);
      if (!busy) {
        /* 收尾：停掉残留的"运行中"动画 */
        for (var i = 0; i < nodes.length; i++) {
          var ai = nodes[i].firstChild;
          if (ai && ai.classList) ai.classList.add("idle");
        }
        /* 只有 onState 到、onMessages 还没到的情况：最后一条若还挂着
           pending，按 busy 直接收尾，别让转圈/光标留在那儿。 */
        var li = lastMsgs.length - 1;
        if (li >= 0 && lastMsgs[li] && lastMsgs[li].pending && nodes[li]) {
          lastMsgs = settlePending(lastMsgs, false);
          try {
            paintNode(nodes[li], withLocal(li, lastMsgs[li]));
            fps[li] = "";
          } catch (e) {}
          snapSchedule(lastMsgs);
        }
      }
    }

    var name = (s.model || "").trim();
    if (name !== lastModel) {
      lastModel = name;
      modelEl.textContent = name || "未选模型";
      modelEl.style.opacity = name ? "" : ".45";
      modelEl.setAttribute("title", name);
    }

    /* 附件：宿主给清单就画芯片（缩略图/文件名/删除），只给个数就退回旧行为 */
    var atts = Array.isArray(s.attachments) ? s.attachments : null;
    var n = atts ? atts.length : (s.pendingAttachments | 0);
    if (n !== lastAtt || atts) {
      lastAtt = n;
      paintAttBar(atts, n);
    }

    /* 待发引用：页面自己记一份（点引用时已知文本），宿主下发了就听宿主的 */
    if (s.pendingQuote != null) {
      setPendingQuote(String(s.pendingQuote || ""), false);
    }
    syncJump();
  };

  /* ── 附件条 ──────────────────────────────────────────────────── */
  var lastAttSig = "", lastAttList = null;
  function paintAttBar(atts, n) {
    var sig = "";
    if (atts) sig = atts.map(function (a) {
      return (a && a.name || "") + "~" + (a && a.path || "");
    }).join("|");
    if (!n && !atts) { attachBar.classList.remove("on"); lastAttList = null; return; }
    if (!n) { attachBar.classList.remove("on"); attsEl.textContent = ""; lastAttSig = ""; lastAttList = null; return; }
    if (atts && sig === lastAttSig) { attachBar.classList.add("on"); return; }
    lastAttSig = sig;
    lastAttList = atts || null;
    attachBar.classList.add("on");
    syncComposerPad();   /* 展开之后再量，垫高才量得准 */
    attsEl.textContent = "";

    if (!atts) {
      /* 老宿主：只有个数 */
      var c0 = el("span", "att");
      var b0 = el("b", null, n + " 个附件待发送");
      c0.appendChild(b0);
      attsEl.appendChild(c0);
      return;
    }
    atts.forEach(function (a, i) {
      a = a || {};
      var chip = el("span", "att");
      chip.dataset.i = i;
      var isImg = !!a.isImage;
      chip.setAttribute("data-img", isImg ? "1" : "0");
      if (isImg && a.path) {
        var th = document.createElement("img");
        th.loading = "lazy";
        th.decoding = "async";
        /* §8.1：file:// 在 about:blank 源下必被拒，缩略图也走 App.image */
        th.onerror = function () {
          th.remove();
          attIcon(chip, 18);
        };
        th.src = "data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==";
        chip.appendChild(th);
        chip.setAttribute("data-act", "viewatt");
        resolveImage(a.path, function (data) {
          if (data) th.src = data;
          else { th.remove(); attIcon(chip, 18); }
        });
      } else {
        attIcon(chip, 15);
        chip.setAttribute("data-act", "viewatt");
      }
      chip.appendChild(el("b", null, a.name || "未命名"));
      var x = el("button", "attx");
      x.type = "button";
      x.setAttribute("data-act", "delatt");
      x.textContent = "×";
      chip.appendChild(x);
      attsEl.appendChild(chip);
    });
  }

  function attIcon(chip, size) {
    var ic = el("span", "ficon");
    ic.innerHTML = size > 16
      ? '<svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4.5" width="18" height="15" rx="2.6"/><circle cx="8.6" cy="9.8" r="1.5"/><path d="M4 16.5l4.6-4.2a1.7 1.7 0 012.3 0L16 17"/></svg>'
      : '<svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M13.5 3H6.8A1.8 1.8 0 005 4.8v14.4A1.8 1.8 0 006.8 21h10.4a1.8 1.8 0 001.8-1.8V7.5z"/><path d="M13.5 3v4.5H19"/></svg>';
    chip.insertBefore(ic, chip.firstChild);
  }

  /* ── 待发引用条 ──────────────────────────────────────────────── */
  var pendingQuoteText = "";
  function setPendingQuoteText(txt) {
    pendingQuoteText = txt || "";
    qbarTxt.textContent = pendingQuoteText;
    qbarEl.classList.toggle("on", !!pendingQuoteText);
    syncComposerPad();
  }
  function setPendingQuote(txt, notify) {
    setPendingQuoteText(txt);
    if (notify && HAS_BRIDGE) call("quote", { index: -1, text: "" });
  }
  qbarX.addEventListener("click", function () { setPendingQuote("", true); });


  window.App.setBusy = function (b) { busy = !!b; setBusyUI(busy); };

  window.App.onSendResult = function (ok, err) {
    if (ok === false && err) toast(err);
  };

  /* ═══ 9. 点击委派：思考 / 工具 / 复制 / 预览 ═══════════════════ */
  listEl.addEventListener("click", function (e) {
    var t = e.target;
    if (!t || !t.closest) return;
    var act = t.closest("[data-act]");
    if (!act) return;
    var kind = act.getAttribute("data-act");

    if (kind === "zoom") {
      var im = act.tagName === "IMG" ? act : act.querySelector("img");
      zoomImg(im ? (im.dataset.src || im.src) : "", im ? im.alt : "");
      return;
    }
    if (kind === "copyfile") { copyPlain(act.dataset.path || "", "路径已复制"); return; }
    if (kind === "openfile") { openLocalPath(act.dataset.path || ""); return; }
    /* 已发送消息里的附件：图片看大图，文件就地读/交给系统 */
    if (kind === "meimg") {
      var shown = act.src && act.src.indexOf("data:") === 0 ? act.src : "";
      if (shown) { zoomImg(shown, act.getAttribute("data-name") || "图片"); return; }
      toast("图片读取中…");
      resolveImage(act.getAttribute("data-path") || "", function (d, err) {
        if (d) zoomImg(d, act.getAttribute("data-name") || "图片");
        else toast(err || "这张图读不到");
      });
      return;
    }
    if (kind === "mefile") {
      var fp2 = act.getAttribute("data-path") || "";
      if (!fp2) { toast("这个附件没有路径"); return; }
      previewFile(fp2, act.getAttribute("data-name") || "");
      return;
    }
    if (kind === "copyurl") { copyPlain(act.dataset.url || "", "地址已复制"); return; }

    if (kind === "reason") {
      var ai = act.closest(".ai");
      if (!ai || !ai.__relBox) return;
      ai.__relBox.hidden = !ai.__relBox.hidden;
      act.classList.toggle("open", !ai.__relBox.hidden);
      return;
    }

    if (kind === "toolsum") {
      var aiS = act.closest(".ai");
      if (!aiS || !aiS.__toolsBox) return;
      aiS.__toolsBox.hidden = !aiS.__toolsBox.hidden;
      act.classList.toggle("open", !aiS.__toolsBox.hidden);
      return;
    }

    if (kind === "tool") {
      var ai2 = act.closest(".ai");
      if (!ai2 || !ai2.__tools) return;
      var idx = act.getAttribute("data-i") | 0;
      ai2.__selTool = (ai2.__selTool === idx) ? -1 : idx;
      for (var k = 0; k < ai2.__tools.children.length; k++) {
        ai2.__tools.children[k].classList.toggle("active", ai2.__selTool === k);
      }
      if (ai2.__selTool === -1) {
        if (ai2.__toolBox) { ai2.__toolBox.remove(); ai2.__toolBox = null; }
      } else {
        var chipT = ai2.__tools.children[ai2.__selTool];
        if (chipT && chipT.__tool) {
          if (!ai2.__toolBox) {
            ai2.__toolBox = el("div", "fold toolfold");
            ai2.__toolBox.appendChild(el("div", "fttl"));
            ai2.__toolsBox.appendChild(ai2.__toolBox);
          }
          ai2.__toolBox.__key = "";
          fillToolBox(ai2, chipT.__tool);
        }
      }
      return;
    }

    var fig = act.closest(".codecard");
    if (!fig) return;
    var codeEl = fig.querySelector("pre code");
    if (!codeEl) return;

    if (kind === "copy") { copyText(codeEl.textContent, act); return; }

    if (kind === "src") {
      var showSrc = fig.classList.toggle("showsrc");
      act.textContent = showSrc ? "预览" : "源码";
      return;
    }

    if (kind === "run") {
      /* html 卡：静态预览 → 沙箱交互（同一套 allow-scripts + CSP 通道，
         跟 card:html 完全一致）。js 卡照旧直接跑。 */
      if (fig.getAttribute("data-kind") === "html") {
        var st = fig.querySelector(".pv:not(.live)");
        if (st) st.remove();
        fig.classList.remove("showsrc");
        /* 不再是静态预览了：换主题时别把它当静态重建 */
        var si = staticPvs.indexOf(fig);
        if (si >= 0) staticPvs.splice(si, 1);
        buildLive(fig, codeEl.textContent, "html");
        act.remove();            /* 已经跑起来了，按钮没用 */
        var srcBtn = fig.querySelector('[data-act="src"]');
        if (srcBtn) srcBtn.textContent = "源码";
      } else {
        runJs(fig, codeEl.textContent, act);
      }
      return;
    }
  });

  function copyText(text, btn) {
    var done = function () {
      var old = btn.textContent;
      btn.textContent = "已复制";
      btn.classList.add("done");
      setTimeout(function () { btn.textContent = old; btn.classList.remove("done"); }, 1300);
    };
    if (navigator.clipboard && navigator.clipboard.writeText) {
      navigator.clipboard.writeText(text).then(done, function () { legacyCopy(text, done); });
    } else {
      legacyCopy(text, done);
    }
  }
  function legacyCopy(text, done) {
    var ta = document.createElement("textarea");
    ta.value = text;
    ta.setAttribute("readonly", "");
    ta.style.cssText = "position:fixed;left:-9999px;top:0;opacity:0";
    document.body.appendChild(ta);
    ta.select();
    ta.setSelectionRange(0, ta.value.length);
    try { document.execCommand("copy"); done(); }
    catch (e) { toast("复制失败，请长按选择"); }
    ta.remove();
  }

  /* ═══ 10. 输入区 ════════════════════════════════════════════════ */
  function setBusyUI(b) {
    if (b) {
      if (!sendEl.classList.contains("stop")) {
        sendEl.classList.add("stop");
        sendEl.innerHTML = ICON_STOP;
        sendEl.setAttribute("aria-label", "停止");
      }
      sendEl.disabled = false;
    } else {
      if (sendEl.classList.contains("stop")) {
        sendEl.classList.remove("stop");
        sendEl.innerHTML = ICON_SEND;
        sendEl.setAttribute("aria-label", "发送");
      }
      sendEl.disabled = inputEl.value.trim() === "";
    }
    syncJump();
  }

  function maxInputH() {
    return Math.max(140, Math.min(220, Math.round((window.innerHeight || 600) * 0.32)));
  }
  function autosize() {
    if (!inputEl.offsetParent) return;
    inputEl.style.height = "auto";
    var h = inputEl.scrollHeight;
    if (!h || h < 64) h = 64;
    inputEl.style.height = Math.min(h, maxInputH()) + "px";
    syncComposerPad();
  }

  function call(name) {
    var args = Array.prototype.slice.call(arguments, 1);
    if (!HAS_BRIDGE) return null;
    try {
      return window.flutter_inappwebview.callHandler.apply(window.flutter_inappwebview, [name].concat(args));
    } catch (e) {
      console.error("callHandler " + name + " failed", e);
      return null;
    }
  }

  /* §8.1：本地图要显示，唯一的正路是 App.image(path) 换 data URI ——
     页面源是 file:// 且 allowFileAccess=false，直接写路径必被拒。
     回调第二参是**失败原因**（宿主现在会说明白：文件没了 / 太大 /
     不在可读范围），调用方决定要不要弹出来。 */
  var IMG_MEMO = {};        /* path → data URI，避免同一张图反复跨桥 */
  var IMG_MEMO_N = 0;
  function resolveImage(path, cb) {
    if (!path) { cb("", "没有路径"); return; }
    var hit = IMG_MEMO[path];
    if (hit) { cb(hit, ""); return; }
    var p = null;
    try { p = call("image", path); } catch (e) { p = null; }
    if (!p || typeof p.then !== "function") { cb("", "宿主没有响应"); return; }
    p.then(function (d) {
      if (typeof d !== "string" || !d) { cb("", "宿主没有返回图片"); return; }
      if (d.charAt(0) === "!") { cb("", d.slice(1)); return; }
      if (IMG_MEMO_N > 40) { IMG_MEMO = {}; IMG_MEMO_N = 0; }
      IMG_MEMO[path] = d; IMG_MEMO_N++;
      cb(d, "");
    }, function () { cb("", "读取失败"); });
  }

  function send() {
    var t = inputEl.value.trim();
    if (!t || busy) return;
    inputEl.value = "";
    autosize();
    setBusyUI(false);
    if (HAS_BRIDGE) call("send", t);
    else demoPush(t);
  }

  sendEl.addEventListener("click", function () {
    if (busy) {
      call("stop");
      if (!HAS_BRIDGE) { busy = false; setBusyUI(false); }
      return;
    }
    send();
  });

  /* ═══ 9.5 @ 工具/插件选择器 ═════════════════════════════════════ */
  var mentionEl = $("mention");
  var mention = { tools: [], plugins: [], at: -1, items: [], cur: 0 };

  window.App.onMentions = function (p) {
    p = p || {};
    mention.tools = p.tools || [];
    mention.plugins = p.plugins || [];
  };

  function mentionQuery(text, caret) {
    var i = text.lastIndexOf("@", Math.max(0, caret - 1));
    if (i < 0) return null;
    var seg = text.slice(i + 1, caret);
    if (!/^[\w\u4e00-\u9fa5-]*$/.test(seg)) return null;
    /* @ 前必须是行首或空白/标点，避免把邮箱当成 mention */
    if (i > 0 && !/[\s（(，。、！？,.(]/.test(text.charAt(i - 1))) return null;
    return { at: i, query: seg };
  }

  function mentionMatches(q) {
    q = q.toLowerCase();
    var out = [];
    function scan(list, kind) {
      for (var i = 0; i < list.length; i++) {
        var t = list[i] || {};
        var id = t.id || "", title = t.title || id;
        if (q && title.toLowerCase().indexOf(q) < 0 && id.toLowerCase().indexOf(q) < 0) continue;
        out.push({ kind: kind, id: id, title: title, note: t.note || "" });
        if (out.length >= 8) return;
      }
    }
    scan(mention.tools, "tool");
    scan(mention.plugins, "plugin");
    return out;
  }

  function closeMention() {
    mention.at = -1;
    mention.items = [];
    mentionEl.hidden = true;
  }
  function openMention(mq) {
    mention.at = mq.at;
    mention.items = mentionMatches(mq.query);
    mention.cur = 0;
    mentionEl.hidden = false;
    mentionEl.textContent = "";
    if (!mention.items.length) {
      mentionEl.appendChild(el("div", "mempty", "没有匹配的工具 / 插件"));
      return;
    }
    for (var i = 0; i < mention.items.length; i++) {
      (function (item, n) {
        var b = el("button", "mitem" + (n === 0 ? " cur" : ""));
        b.type = "button";
        b.appendChild(el("span", "ic", item.kind === "tool" ? "T" : "P"));
        var tx = el("span", "tx");
        tx.appendChild(el("div", "tt", item.title));
        if (item.note) tx.appendChild(el("div", "nt", item.note));
        b.appendChild(tx);
        b.addEventListener("click", function () { applyMention(item); });
        mentionEl.appendChild(b);
      })(mention.items[i], i);
    }
  }
  function refreshMention() {
    if (mention.at < 0) return;
    var mq = mentionQuery(inputEl.value, inputEl.selectionStart || 0);
    if (mq) openMention(mq); else closeMention();
  }
  function applyMention(item) {
    var caret = inputEl.selectionStart || 0;
    var token = "@" + item.kind + ":" + item.id + " ";
    inputEl.value = inputEl.value.slice(0, mention.at) + token +
                    inputEl.value.slice(caret);
    var pos = mention.at + token.length;
    closeMention();
    inputEl.focus();
    inputEl.selectionStart = inputEl.selectionEnd = pos;
    autosize();
    setBusyUI(busy);
  }

  inputEl.addEventListener("input", function () {
    autosize();
    if (!busy) sendEl.disabled = inputEl.value.trim() === "";
    var mq = mentionQuery(inputEl.value, inputEl.selectionStart || 0);
    if (mq && (mention.tools.length || mention.plugins.length)) openMention(mq);
    else closeMention();
  });

  inputEl.addEventListener("keydown", function (e) {
    var open = !mentionEl.hidden && mention.items.length;
    if (open && (e.key === "ArrowDown" || e.key === "ArrowUp")) {
      e.preventDefault();
      mention.cur = (mention.cur + (e.key === "ArrowDown" ? 1 : mention.items.length - 1)) % mention.items.length;
      var kids = mentionEl.querySelectorAll(".mitem");
      for (var i = 0; i < kids.length; i++) kids[i].classList.toggle("cur", i === mention.cur);
      var c = kids[mention.cur];
      if (c && c.scrollIntoView) c.scrollIntoView({ block: "nearest" });
      return;
    }
    if (open && (e.key === "Enter" || e.key === "Tab") && !e.isComposing) {
      e.preventDefault();
      applyMention(mention.items[mention.cur]);
      return;
    }
    if (open && e.key === "Escape") { e.preventDefault(); closeMention(); return; }

    if (e.key === "Enter" && !e.shiftKey && !e.isComposing) {
      e.preventDefault();
      send();
    }
  });

  $("bFile").addEventListener("click", function () { call("pickFile"); });
  $("bImg").addEventListener("click", function () { call("pickImage"); });
  modelEl.addEventListener("click", function () { call("pickModel"); });

  /* 空白态的大标题固定成站点标语；时段问候挪到上面那行等宽 HUD 里，
     不然首页标题每隔几小时就自己改名，像页面在乱动。 */
  (function () {
    var h = new Date().getHours();
    var slot = h >= 5 && h < 11 ? "MORNING SHIFT / 早上好" :
               h >= 11 && h < 13 ? "NOON SHIFT / 中午好" :
               h >= 13 && h < 18 ? "AFTERNOON SHIFT / 下午好" :
                                   "NIGHT SHIFT / 晚上好";
    var k = $("ekick");
    if (k) k.textContent = slot;
  })();

  /* 键盘弹出 / 旋转屏幕后重新量一次输入框 ——
     注意别再挂在这里逐帧量：autosize 读 scrollHeight + 写 height 是同步布局，
     键盘动画里每帧来一次，就是卡顿的大头。统一交给 settleViewport（见 §6 视口段）。 */

  /* ═══ 10.5 选择 / 长按菜单：复制 · 粘贴 · 全选 · 标记 · 引用 ═══ */
  var menuEl = null, menuBorn = 0;
  function hideMenu() {
    if (menuEl) { menuEl.remove(); menuEl = null; }
  }
  function showMenu(items, x, y) {
    hideMenu();
    menuEl = el("div", "cmenu");
    for (var i = 0; i < items.length; i++) {
      (function (it) {
        var b = el("button", it.danger ? "danger" : null, it.label);
        b.addEventListener("click", function () { hideMenu(); it.run(); });
        menuEl.appendChild(b);
      })(items[i]);
    }
    document.body.appendChild(menuEl);
    var r = menuEl.getBoundingClientRect();
    var left = Math.max(8, Math.min(x - r.width / 2, window.innerWidth - r.width - 8));
    var top = y - r.height - 12;
    if (top < 8) top = Math.min(y + 20, window.innerHeight - r.height - 8);
    menuEl.style.left = left + "px";
    menuEl.style.top = top + "px";
    menuBorn = Date.now();
  }
  function copyPlain(text, okMsg) {
    copyTextSilent(text, function () { toast(okMsg || "已复制"); });
  }
  function copyTextSilent(text, done) {
    if (navigator.clipboard && navigator.clipboard.writeText) {
      navigator.clipboard.writeText(text).then(done, function () { legacyCopy(text, done); });
    } else {
      legacyCopy(text, done);
    }
  }
  function currentSel() {
    var s = window.getSelection();
    if (!s || s.rangeCount === 0 || s.isCollapsed) return null;
    var txt = s.toString();
    if (!txt.trim()) return null;
    var range = s.getRangeAt(0);
    var rect = range.getBoundingClientRect();
    var node = range.startContainer;
    var elm = node.nodeType === 1 ? node : node.parentElement;
    var row = elm && elm.closest ? elm.closest(".row") : null;
    return { text: txt, rect: rect, row: row };
  }
  function rowIndex(row) {
    for (var i = 0; i < nodes.length; i++) if (nodes[i] === row) return i;
    return -1;
  }
  function addLocalMark(idx, text) {
    var base = lastMsgs[idx];
    if (!base) return;
    var content = base.content || "";
    if (content.indexOf(text) < 0) { toast("选区和原文对不上，没法标记"); return; }
    var arr = localMarks[idx] || (localMarks[idx] = []);
    var used = {};
    (base.marks || []).concat(arr).forEach(function (k) {
      if (k.text === text) used[k.occurrence | 0] = 1;
    });
    var occ = 0, pos = 0, n = 0;
    while ((pos = content.indexOf(text, pos)) >= 0) {
      if (!used[n]) { occ = n; break; }
      n++; pos += text.length;
      occ = n;
    }
    arr.push({ text: text, occurrence: occ, color: "yellow", kind: "mark" });
    /* 强制该条重画：清掉正文缓存指纹 */
    var row = nodes[idx];
    if (row) {
      var ai = row.firstChild;
      if (ai && ai.__body) { ai.__body.__src = ""; ai.__body.__marks = ""; }
      else if (row.__me) row.__me.__sig = "";
      paintNode(row, withLocal(idx, base));
    }
    toast("已标记（本页内有效）");
  }
  function clearLocalMark(idx) {
    localMarks[idx] = [];
    var row = nodes[idx];
    if (row) {
      var ai = row.firstChild;
      if (ai && ai.__body) { ai.__body.__src = ""; ai.__body.__marks = ""; }
      else if (row.__me) row.__me.__sig = "";
      paintNode(row, lastMsgs[idx]);
    }
    toast("已清除本页标记");
  }
  /* 引用：挂在输入框上方的待发条里，不往输入框塞 "> "；
     宿主在发送时把 quote 附到消息上（onQuote 通道） */
  function quoteText(txt, idx) {
    var t = String(txt || "").trim();
    if (!t) return;
    setPendingQuoteText(t);
    if (HAS_BRIDGE) {
      call("quote", { index: typeof idx === "number" ? idx : -1, text: t });
    }
    inputEl.focus();
  }
  /* 读剪贴板：**三道路径按可靠性排序**
     ① 宿主 handler `paste`（Android WebView 里唯一稳定的一条 ——
        页面是 about:blank 源，navigator.clipboard 要安全上下文 + 授权，
        而 `disableContextMenu` 又把长按菜单关了）；
     ② navigator.clipboard.readText（桌面预览 / 新内核可能给）；
     ③ 都拿不到就提示用输入法粘贴，绝不假装成功。 */
  function readClipboard() {
    var host = null;
    if (HAS_BRIDGE) {
      try {
        host = (window.App && typeof window.App.paste === "function")
          ? window.App.paste() : call("paste");
      } catch (e) { host = null; }
    }
    if (!host || typeof host.then !== "function") {
      host = Promise.resolve(host == null ? "" : String(host));
      /* 宿主这一路返回空串时，再看看浏览器接口（桌面预览用得上） */
      return host.then(function (t) {
        if (t) return t;
        if (navigator.clipboard && navigator.clipboard.readText) {
          return navigator.clipboard.readText().then(function (x) { return x || ""; },
                                                     function () { return ""; });
        }
        return "";
      });
    }
    return host.then(function (t) { return t == null ? "" : String(t); },
                     function () { return ""; });
  }
  function insertAtCaret(t) {
    var st = inputEl.selectionStart, en = inputEl.selectionEnd;
    if (typeof st !== "number" || st < 0) st = inputEl.value.length;
    if (typeof en !== "number" || en < st) en = st;
    inputEl.value = inputEl.value.slice(0, st) + t + inputEl.value.slice(en);
    var pos = st + t.length;
    try { inputEl.selectionStart = inputEl.selectionEnd = pos; } catch (e) {}
    autosize();
    setBusyUI(busy);
  }
  function pasteIntoInput() {
    readClipboard().then(function (t) {
      if (!t) { toast("剪贴板里没有文字"); return; }
      inputEl.focus();
      insertAtCaret(t);
    });
  }
  function selMenu(sel, x, y) {
    var items = [{ label: "复制", run: function () { copyPlain(sel.text); } }];
    var idx = sel.row ? rowIndex(sel.row) : -1;
    if (idx >= 0 && lastMsgs[idx]) {
      var hasLocal = localMarks[idx] && localMarks[idx].length;
      items.push({ label: "全选本条", run: function () {
        var r = document.createRange();
        r.selectNodeContents(sel.row);
        var s = window.getSelection();
        s.removeAllRanges(); s.addRange(r);
      }});
      if (lastMsgs[idx].content && lastMsgs[idx].content.indexOf(sel.text.trim()) >= 0) {
        items.push({ label: "标记", run: function () { addLocalMark(idx, sel.text.trim()); }});
      }
      if (hasLocal) items.push({ label: "清标记", run: function () { clearLocalMark(idx); }});
      items.push({ label: "引用", run: function () { quoteText(sel.text, idx); }});
    }
    showMenu(items, x, y);
  }
  function wholeMenu(row, x, y) {
    var idx = rowIndex(row);
    var m = idx >= 0 ? lastMsgs[idx] : null;
    var items = [];
    if (m && (m.content || "").trim()) {
      items.push({ label: "复制全文", run: function () { copyPlain(m.content.trim()); }});
      items.push({ label: "引用整条", run: function () { quoteText(m.content.trim(), idx); }});
    }
    if (!items.length) return;
    showMenu(items, x, y);
  }
  function inputMenu(x, y) {
    var items = [{ label: "粘贴", run: pasteIntoInput }];
    var hasSel = inputEl.selectionStart !== inputEl.selectionEnd;
    items.push({ label: hasSel ? "剪切" : "全选", run: function () {
      if (hasSel) {
        copyPlain(inputEl.value.slice(inputEl.selectionStart, inputEl.selectionEnd));
        var st = inputEl.selectionStart;
        inputEl.value = inputEl.value.slice(0, st) + inputEl.value.slice(inputEl.selectionEnd);
        inputEl.selectionStart = inputEl.selectionEnd = st;
        autosize();
      } else {
        inputEl.focus(); inputEl.select();
      }
    }});
    if (hasSel) items.push({ label: "复制", run: function () {
      copyPlain(inputEl.value.slice(inputEl.selectionStart, inputEl.selectionEnd));
    }});
    showMenu(items, x, y);
  }

  /* 长按检测：520ms 没动就算长按 */
  var pressTimer = 0, pressX = 0, pressY = 0;
  function armPress(x, y) {
    pressX = x; pressY = y;
    clearTimeout(pressTimer);
    pressTimer = setTimeout(function () {
      pressTimer = 0;
      var sel = currentSel();
      if (sel && sel.rect && sel.rect.width) { selMenu(sel, x, sel.rect.top); return; }
      var t = document.elementFromPoint(x, y);
      var row = t && t.closest ? t.closest(".row") : null;
      if (row) wholeMenu(row, x, y);
    }, 520);
  }
  function disarmPress(moved, x, y) {
    if (!pressTimer) return;
    if (moved && (Math.abs(x - pressX) > 10 || Math.abs(y - pressY) > 10)) {
      clearTimeout(pressTimer);
      pressTimer = 0;
    }
  }
  listEl.addEventListener("touchstart", function (e) {
    if (e.touches.length !== 1) return;
    armPress(e.touches[0].clientX, e.touches[0].clientY);
  }, { passive: true });
  listEl.addEventListener("touchmove", function (e) {
    if (e.touches.length === 1) disarmPress(true, e.touches[0].clientX, e.touches[0].clientY);
  }, { passive: true });
  listEl.addEventListener("touchend", function () { clearTimeout(pressTimer); pressTimer = 0; });
  listEl.addEventListener("touchcancel", function () { clearTimeout(pressTimer); pressTimer = 0; });

  /* 桌面/部分 WebView 会发 contextmenu（长按也走这里）*/
  listEl.addEventListener("contextmenu", function (e) {
    var sel = currentSel();
    if (sel) { e.preventDefault(); selMenu(sel, e.clientX, e.clientY); return; }
    var row = e.target && e.target.closest ? e.target.closest(".row") : null;
    if (row) { e.preventDefault(); wholeMenu(row, e.clientX, e.clientY); }
  });
  inputEl.addEventListener("contextmenu", function (e) {
    e.preventDefault();
    inputMenu(e.clientX, e.clientY);
  });
  inputEl.addEventListener("touchstart", function (e) {
    if (e.touches.length !== 1) return;
    var x = e.touches[0].clientX, y = e.touches[0].clientY;
    clearTimeout(pressTimer);
    pressTimer = setTimeout(function () {
      pressTimer = 0;
      inputMenu(x, y);
    }, 520);
  }, { passive: true });
  inputEl.addEventListener("touchend", function () { clearTimeout(pressTimer); pressTimer = 0; });

  /* 点别处 / 滚动 → 收起菜单（刚弹出的 400ms 内不响应，防误触）*/
  document.addEventListener("click", function (e) {
    if (!menuEl) return;
    if (Date.now() - menuBorn < 400) return;
    if (menuEl.contains(e.target)) return;
    hideMenu();
  }, true);
  listEl.addEventListener("scroll", hideMenu, { passive: true });

  function toast(msg) {
    var t = el("div", null, msg);
    t.style.cssText = "position:fixed;left:16px;right:16px;bottom:calc(92px + env(safe-area-inset-bottom));" +
      "padding:10px 13px;background:var(--dph-paper,#f2f3ef);color:var(--dph-ink-dark,#151717);" +
      "border-left:4px solid var(--dph-signal,#f2f500);border-radius:0;" +
      "font:700 11.5px/1.55 var(--mono,monospace);letter-spacing:.04em;z-index:9;" +
      "box-shadow:0 10px 24px rgba(0,0,0,.5);opacity:0;" +
      "transition:opacity .2s ease;pointer-events:none";
    document.body.appendChild(t);
    requestAnimationFrame(function () { t.style.opacity = "1"; });
    setTimeout(function () {
      t.style.opacity = "0";
      setTimeout(function () { t.remove(); }, 240);
    }, 2600);
  }

  autosize();
  setBusyUI(false);

  /* 先把上一次的会话画出来，再接宿主的推送 ——
     页面重载（换深浅色、换主题包、系统回收）后不会出现"聊天没了"的空窗 */
  try { restoreSnapshot(); } catch (e) {}

  /* ═══ 11. 浏览器预览 demo（宿主里不执行）════════════════════════ */
  var demoCount = 0;

  var DEMO_HTML = '<div style="font-family:-apple-system;\n  padding:18px;background:#0f172a;color:#e2e8f0">\n  <div style="font-size:12px;letter-spacing:.2em;opacity:.5">\n    TODAY\n  </div>\n  <div style="font-size:44px;margin:6px 0 2px">21°</div>\n  <div style="opacity:.6;font-size:13px">多云 · 微风</div>\n</div>';

  var demoMsgs = [
    { role: "user", content: "写个 HTML 卡片展示今日天气，顺便说下你的思路。",
      reasoning: "", reasoningMs: 0, pending: false, error: null, tools: [],
      marks: [], quote: null },
    { role: "assistant",
      content: "先确认一件事：这张卡片会跑在沙箱 iframe 里，脚本被禁用，所以只用 CSS 静态排版。\n\n```html\n" + DEMO_HTML + "\n```\n\n关键取舍：\n\n- 温度用 `44px` 一招压住视觉重心，其余全部降级为辅助信息\n- 不用图片，深色底 + 字色对比度直接达标\n- 缩到 320px 宽也不会折行\n\n> 想换成浅色，只改 background 和 color 两行即可。\n\n| 元素 | 字号 | 作用 |\n|---|---|---|\n| 温度 | 44px | 视觉锚点 |\n| 天气 | 13px | 次要说明 |\n\n再补几类卡片：\n\n```card:html\n<div style=\"display:flex;gap:8px;align-items:center\">\n  <div style=\"width:38px;height:38px;border-radius:50%;background:#2E4B45;color:#fff;display:flex;align-items:center;justify-content:center;font-size:17px\">✓</div>\n  <div>\n    <div style=\"font-weight:600\">交互卡片</div>\n    <div style=\"opacity:.6;font-size:12px\">在沙箱里跑 HTML+JS，高度自适应</div>\n  </div>\n</div>\n<script>\nvar b=document.createElement('button');\nb.textContent='点我 '+(window.__n||0);\nb.style.cssText='margin-top:8px;padding:6px 12px;border-radius:8px;border:1px solid #ccc;background:#fff';\nb.onclick=function(){window.__n=(window.__n||0)+1;b.textContent='点我 '+window.__n};\ndocument.getElementById('rt').appendChild(b);\n</" + "script>\n```\n\n```js\nconst f = (n) => n * 2;\nconsole.log('结果', f(21));\nconsole.log({ ok: true, cards: ['html', 'js', 'mindmap'] });\n```\n\n```mindmap\nmindmap\n  root((主题包))\n    结构\n      消息列表\n      输入框\n      工具行\n    能力\n      标记与引用\n      卡片\n        HTML\n        JS\n        思维导图\n```\n\n正文里出现网址也会自动补卡片：文档在 https://registry.npmmirror.com/web/detail/mermaid 可以查，生成的文件在 D:\\pj3\\theme-mojian.html。",
      reasoning: "用户要一张天气卡片。先判断运行环境——主题包页面不能联网，所以必须给静态 HTML。\n再判断尺寸：手机竖屏，卡片宽度约 340px，温度 44px 够不够？够，再大会和标题打架。",
      reasoningMs: 4200, pending: false, error: null,
      marks: [
        { text: "静态排版", occurrence: 0, color: "yellow", kind: "mark" },
        { text: "对比度直接达标", occurrence: 0, color: "green", kind: "mark" },
        { text: "只改 background 和 color 两行即可", occurrence: 0, color: "blue", kind: "quote" }
      ],
      quote: null,
      tools: [
        { name: "read_file", summary: "read_file", status: "success", args: '{"path":"ui/chat.dart","lines":120}', output: "已读取 412 行", error: false },
        { name: "python", summary: "python", status: "success", args: '{"code":"print(21*1.0)"}', output: "21.0", error: false }
      ] },
    { role: "user", content: "引用刚才那句关于对比度的话，解释一下为什么。",
      reasoning: "", reasoningMs: 0, pending: false, error: null, tools: [],
      marks: [],
      quote: { text: "不用图片，深色底 + 字色对比度直接达标", color: "green" } }
  ];

  function demoPush(text) {
    var all = demoMsgs.slice();
    all.push({ role: "user", content: text, reasoning: "", reasoningMs: 0,
               pending: false, error: null, tools: [], marks: [], quote: null });
    window.App.onMessages(all, true);
    window.App.onState({ busy: true, model: "deepseek-v4-flash", canStop: true, pendingAttachments: 0 });

    var reply = "收到。这一版补上了标记与引用，也顺手压了流式渲染的开销：\n\n" +
      "1. **标记**：按「原文 + 第几次出现」定位，荧光笔半透明，引用是虚线\n" +
      "2. **引用**：`quote` 渲染成带色条的卡片，气泡上方独立一行\n" +
      "3. **性能**：推送用 rAF 合并，正文只重画还没定稿的尾部\n" +
      "4. **收尾**：思考结束后不再残留转圈和光标\n\n" +
      "顺手验证一下公式：行内质能方程 $E = mc^2$，块级的：\n\n" +
      "$$\n\\sum_{i=1}^{n} i = \\frac{n(n+1)}{2} \\approx \\int_0^n x\\,dx\n$$\n\n" +
      "```js\n// 已定稿的块不再重画\nbody.appendChild(renderMD(content.slice(stableEnd), { base: stableEnd }));\n```\n\n" +
      "> 真机上这些全部来自宿主推送。";
    var i = 0;
    var timer = setInterval(function () {
      i = Math.min(reply.length, i + 14);
      var cur = all.slice();
      cur.push({
        role: "assistant", content: reply.slice(0, i),
        reasoning: "增量渲染的关键是给每个顶层块记下结束偏移；只有偏移早于上次长度的块才算定稿。",
        reasoningMs: 1600, pending: i < reply.length, error: null,
        marks: [{ text: "荧光笔半透明", occurrence: 0, color: "pink", kind: "mark" }],
        quote: null,
        tools: [{ name: "write_file", summary: "write_file",
                  status: i < reply.length ? "running" : "success",
                  args: '{"path":"theme-mojian.html"}', output: "", error: false }]
      });
      window.App.onMessages(cur, i < reply.length);
      if (i >= reply.length) {
        clearInterval(timer);
        window.App.onState({ busy: false, model: "deepseek-v4-flash", canStop: false, pendingAttachments: 0 });
      }
    }, 30);
  }

  if (!HAS_BRIDGE) {
    $("bFile").addEventListener("click", function () {
      demoCount++;
      window.App.onState({ busy: busy, model: "deepseek-v4-flash", canStop: busy, pendingAttachments: demoCount });
    });
    $("bImg").addEventListener("click", function () {
      demoCount++;
      window.App.onState({ busy: busy, model: "deepseek-v4-flash", canStop: busy, pendingAttachments: demoCount });
    });

    /* 预览态也只走终末地这一套冷黑，不再跟随系统深浅 */
    var PALETTES = {
      only: {
        primary: "#f2f500", onPrimary: "#101313", surface: "#151a1b",
        page: "#0a0d0e", text: "#eef3ef", muted: "#89938f", bubble: "#202728",
        dark: "false", fontScale: "1.0"
      }
    };
    function applyDemoPalette() {
      window.App.onTheme(PALETTES.only);
    }
    applyDemoPalette();

    window.App.onMentions({
      tools: [
        { id: "python", title: "Python 运行", note: "执行一段 Python 并返回输出" },
        { id: "termux", title: "Termux", note: "在手机终端里跑命令" },
        { id: "web", title: "网页抓取", note: "读取一个网页的正文" },
        { id: "file_op", title: "文件操作", note: "读写本机文件" }
      ],
      plugins: [{ id: "weather", title: "天气查询", note: "按城市查当前天气" }]
    });

    /* 预览模式：长按模型名切深浅色（真机上这里会打开原生模型选择器） */
    (function () {
      var tmr = 0;
      var start = function () { tmr = setTimeout(function () {
        applyDemoPalette();
        toast("终末地主题只提供这一套冷黑配色");
        tmr = -1;
      }, 520); };
      var stop = function () { if (tmr > 0) clearTimeout(tmr); };
      modelEl.addEventListener("pointerdown", start);
      modelEl.addEventListener("pointerup", stop);
      modelEl.addEventListener("pointercancel", stop);
      modelEl.addEventListener("click", function () {
        if (tmr === -1) { tmr = 0; return; }
        toast("预览模式下没有原生模型选择器");
      });
    })();

    window.App.onState({
      busy: false, model: "deepseek-v4-flash", canStop: false,
      pendingAttachments: 2,
      attachments: [
        { name: "weather-card.html", path: "", isImage: false },
        { name: "screenshot.png", path: "", isImage: true }
      ]
    });
    setTimeout(function () {
      window.App.onMessages(demoMsgs, false);
    }, 60);
  }
})();
</script>
<script>
(function () {
  /* 安全区兜底：有些 WebView 的 env(safe-area-inset-top) 恒为 0，
     而页面又是从状态栏下面铺满的 —— 那就补一个常见状态栏高度。 */
  var probe = document.createElement("div");
  probe.style.cssText = "position:fixed;top:0;left:0;width:1px;height:env(safe-area-inset-top,0px);opacity:0;pointer-events:none";
  document.body.appendChild(probe);
  var h = probe.getBoundingClientRect().height;
  probe.remove();
  if (!(h > 1)) {
    document.documentElement.style.setProperty("--st", "26px");
  }
})();
</script>
</body>
</html>
