---
title: "統一介面覆蓋異質可靠性域的結構性矛盾"
date: 2026-05-16
source: tech
---

- **統一介面覆蓋異質可靠性域的結構性矛盾**：當一層 CLI interface 同時覆蓋確定性 adapter（HTTP parsing → stdout，失敗可枚舉：timeout、403、parse error）和 agent automation（CDP → DOM interaction，失敗是情境依賴的開放集合），兩者不可能共享同一套錯誤處理策略。判斷信號：當 skill 文件需要用自然語言教 agent 如何 recover from broken page state，說明錯誤空間已超出結構化 error handling 的能力範圍——你在用文檔補償類型系統無法表達的東西。
