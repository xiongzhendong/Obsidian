### 研究背景
![[Pasted image 20261030215058.png]]

城市导航要求智能体根据局部、第一人称视角连续做出动作，但目的地往往相距较远，中途还要经过路口或转弯。

已有导航推理方法虽会利用语言模型分析观察和历史，部分方法也会预测后续行动或地标，但
1. 历史上下文记录了过去的观察和行动，但对之后如何朝长期目标推进提供的指导有限。因此，局部决策单独来看或许合理，却可能在不同步骤之间缺乏一致性。
2. 用语言的方式推理未来动作和地标不能提供动作执行后带来的视觉观测。

### 主要贡献
Our central idea is to maintain medium-horizon subgoals to guide local decisions, evaluate proposed actions using predicted visual outcomes before execution, and update navigation memory using execution feedback.

- PreAct-Nav, a training-free navigation harness that reviews and revises policy actions using memory and visual prediction before execution.

- Navigation Memory maintains a medium-horizon subgoal and observed experience; real execution feedback updates memory for subsequent decisions.

- an action-conditioned world model as a Predictive Sandbox, providing future visual cues for action reasoning under the current subgoal.

### 研究方法

![[Pasted image 20261030223153.png]]

**Navigation task and policy actions.**
$$(a_t^0,p_t^\mathrm{stop})=\pi(g,\mathcal{I}_t^\pi,\mathbf{p}_t) \\ a_t^0=(\mathbf{w}_{t,1}^0,\ldots,\mathbf{w}_{t,K}^0)$$
**Reasoning context.**
$$\mathcal{C}_t=(g,\mathcal{I}_t,\mathcal{M}_t,a_t^0),\quad\mathcal{M}_t=(E_t,S_t),$$
The reasoning history $T^j_t = ((a^i_t, r^i_t))_{i<j}$ retains earlier actions and reviews, with $T^0_t = ∅$. 

---
#### NAVIGATION MEMORY
Experience $E_t$ records past outcomes; the current subgoal $S_t$ guides action review and revision.

**Observed experience.**  Each record contains Current observation, Executed action, Next observation, and Observed feedback. Recent images and concise summaries are retained in temporal order to identify actions that failed to produce expected progress. Past actions use the local coordinate frame of their associated observations.

**Current subgoal.** A subgoal specifies an intermediate objective across local actions:
$$S_t=(z_t,c_t,q_t,\sigma_t),$$
The subgoal conditions the reasoner; the policy remains conditioned on the final destination g.

**State update from execution feedback.** After executing $a_t$, the reasoner compares the next real observation with the Expected future cues $ν_t$ retained from action review:
$$v_{t+1}=\mathcal{R}_{\mathrm{update}}(g,S_t,a_t,\nu_t,o_t,o_{t+1}).$$
Feedback $v_{t+1}$ reports SUPPORTED, ACHIEVED, CONTRADICTED, or UNRESOLVED, with a brief explanation of open-question progress. The observed transition is appended to experience, and the feedback updates the subgoal:$$\begin{aligned}&E_{t+1}=\mathrm{Select}\left(E_{t}\parallel(e_{t})\right),\\&S_{t+1}=\mathrm{Update}(S_{t},v_{t+1}),\quad\mathcal{M}_{t+1}=(E_{t+1},S_{t+1}).\end{aligned}$$Supported or unresolved feedback preserves the active objective; achievement or contradiction closes it before a new objective is established. Only real transitions enter observed experience

---
#### PREDICTIVE SANDBOX

**Action-conditioned prediction.**
$$Q_t^j=(o_t,\mathbf{C}(a_t^j),\ell,\epsilon),\quad\widehat{O}_t^j=\mathcal{W}(Q_t^j)=(\widehat{o}_{t,1}^j,\ldots,\widehat{o}_{t,L}^j),$$
Across revisions, the initial observation, prompt, and seed remain fixed; only the proposed movement changes.

**Visual evidence collection.** The reasoner inspects the rollout and its action using the destination, observations, subgoal, and open question. It returns three fields: **Prediction quality** assesses coherence, movement consistency, and relevance; **Rationale** links visual cues to the action assessment; and **Expected future cues** specify changes to check after execution.

---
#### AGENTIC REASONING HARNESS
Each decision permits at most B reviewed actions, including the policy action, and B − 1 revisions.

**Stage I: Action Preview.** It requests a preview when the action assessment is unresolved and a forecast could supply missing evidence. 
$$\lambda_t^j=\mathbf{1}\left[\bar{\omega}_t^j=\text{UNRESOLVED}\wedge\bar{\kappa}_t^j=1\right]$$
where ¯ωj t is the observation-based action assessment and ¯κj t indicates whether a preview would help resolve the open question.

**Stage II: Action Review.** The reasoner reviews the proposed action against the current subgoal, incorporating the predicted rollout when available:
$$r_t^j=(\omega_t^j,\chi_t^j,\rho_t^j,\nu_t^j,\kappa_t^j)=\mathcal{R}_\mathrm{review}(\mathcal{C}_t,\mathcal{T}_t^j,a_t^j,\widehat{O}_t^j).$$
The action assessment $ω_j$ t is SUPPORTED, CONTRADICTED, or UNRESOLVED. The remaining entries are prediction quality $χ^j_ t$ , rationale $ρ^j_ t$ , expected future cues $ν^j _t$ , and preview usefulness $κ^j _t$. Without a rollout, prediction quality is not assessed; other fields use observations and memory. A supported assessment accepts the action. A contradicted assessment identifies a reason to change it:
$$\begin{aligned}(a_{t}^{j+1},\rho_{t}^{j+1},\nu_{t}^{j+1})&=\mathcal{R}_{\mathrm{revise}}(\mathcal{C}_{t},\mathcal{T}_{t}^{j},a_{t}^{j},r_{t}^{j},\widehat{O}_{t}^{j}),\\&\mathrm{if~}\omega_{t}^{j}=\text{CONTRADICTED }\wedge j+1<B.\end{aligned}$$
**Stage III: Action Execute.** The agent executes the first supported action.  If no action is supported, it retains the policy action. 

**Stage IV: Memory Update.** The reasoner compares the next real observation with the executed action’s expected cues using Equation 3. Equation 4 updates experience and subgoal progress.


### 实验

#### Comparison 
![[Pasted image 20261031173428.png]]
![[Pasted image 20261031173450.png]]
**主要结果：** 在 UrbanNav 未见测试集上，使用 Qwen 推理器时，成功率（SR）由原始策略的 **78.14%** 提升至 **83.38%**，路径效率指标 SPL 从 **74.73%** 提升至 **77.87%**，导航误差则由 **5.9659 米** 降至 **5.3353 米**。在 CityWalker 上，SR 达到 **79.22%**；换用 NoMaD 和 OmniVLA 两种冻结导航策略时，SR 分别提升 **2.78** 和 **3.58 个百分点**，显示该框架不只适用于单一策略。

---
#### Ablation 
![[Pasted image 20261031173605.png]]
消融实验表明，记忆和预测沙盒各自都有贡献：只用推理框架时 SR 为 **80.56%**，加入记忆后为 **82.37%**，完整系统达到 **83.38%**。预测带来的收益在更长、转弯更多的路线中通常更明显。

**局限与判断：** 关键瓶颈是预测视频可能含糊或不准确，而且生成成本高：论文报告，生成一段含 60 个预测帧的视频平均需要 **11.37 秒**。因此，这项工作的价值主要在于展示一种把“中期目标、动作预演、真实反馈”串成闭环的导航框架；若用于对时延敏感的实际导航，还需要更快、更可靠的世界模型。