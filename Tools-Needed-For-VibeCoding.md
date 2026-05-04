
**核心** 是如何在Vibe Coding过程中给开发者提供**确定性**、**可溯源**

- [ ] 代码与文档内容一致性校验Agent
- [ ] 代码bug扫描Agent
	- FM-Agent：针对整个codebase，用于阶段性扫描（因为token小号高）
		- 缺点：细粒度（需要别的工具补充）
		- [ ] 研究FM-Agent代码
		- [ ] 本地部署FM-Agent服务
	- [ ] 整个codebase粒度的阶段性扫描Agent工具
	- [ ] 增量小扫描的Agent工具
- [ ] 类似OpenSpec这样的开发**迭代**工具，来提供**可溯源**
	- ？OpenSpec是不是太不灵活了
	- [ ] 研究OpenSpec代码
