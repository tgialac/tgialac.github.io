---
title: "Tokens for Thought"
date: 2026-07-24T09:00:00+07:00
tags: ["ai", "llm", "economics", "inference", "agents", "systems"]
math: true
summary: "Token không phải là giá trị. Nó là đồng hồ đo cho một hệ thống biến compute, bộ nhớ, độ trễ, quyền hành động và rủi ro thành một quyết định. AI tokenomics là cách thiết kế đồng hồ đo đó cho đúng."
---

> *Token is not the unit of intelligence. It is the meter on an intelligent production system.*

Một agent có thể giải được một bài toán khó với giá vài xu. Nó cũng có thể đốt hàng triệu token để viết ra một câu trả lời không ai dùng. Cả hai đều có cùng đơn vị kế toán: token. Nhưng chỉ một trong hai tạo ra giá trị.

Đó là lý do “AI tokenomics” đáng được xem như một ngành thiết kế hệ thống, không chỉ là bảng giá API. Nó hỏi những câu lớn hơn:

- Một token thật sự tốn gì để phục vụ—không chỉ giá niêm yết, mà còn GPU, bộ nhớ KV, mạng, năng lượng và công suất dự phòng?
- Nên bỏ thêm bao nhiêu compute vào *một* câu hỏi trước khi lợi ích biên biến mất?
- Vì sao một mô hình nhỏ, một router, hay một cache đôi khi có giá trị kinh tế lớn hơn mô hình mạnh nhất?
- Ai trả tiền cho reasoning ẩn, retry, context phình to và những vòng trao đổi vô ích giữa agents?
- Khi giá token giảm rất nhanh, lợi thế cạnh tranh chuyển từ “mua token rẻ” sang đâu?

Luận điểm của bài này đơn giản:

> **Token là một đầu vào sản xuất có giá; giá trị của nó chỉ xuất hiện khi token thay đổi một quyết định, một hành động, hoặc một rủi ro theo hướng tốt hơn.**

Nói cách khác, token không phải tiền. Token giống kilowatt-giờ hơn: một đơn vị đo tiêu thụ có thể rất rẻ, rất đắt, hoặc hoàn toàn lãng phí tùy hệ thống biến nó thành kết quả.

## Bản đồ: từ token đến giá trị

Một lần gọi LLM nằm trên ít nhất năm lớp. Bỏ qua bất kỳ lớp nào sẽ cho ra một “tối ưu” giả.

| Lớp | Câu hỏi kinh tế | Sai lầm thường gặp |
| --- | --- | --- |
| **Token** | Bao nhiêu token vào, ra, reasoning và tool context? | Chỉ đếm output hiển thị cho người dùng. |
| **Hệ thống** | GPU, KV cache, batch, mạng và SLO biến token thành chi phí/độ trễ thế nào? | Đồng nhất token với FLOPs hoặc với latency. |
| **Mô hình** | Mô hình/cascade nào đạt chất lượng cần thiết với chi phí thấp nhất? | Luôn gọi model mạnh nhất. |
| **Workflow** | Token ở bước nào thay đổi xác suất thành công của công việc? | Tối ưu cost per request thay vì cost per successful outcome. |
| **Thị trường & quản trị** | Giá bán, quyền lực định giá, dữ liệu, lock-in, rủi ro và trách nhiệm được phân bổ ra sao? | Coi bảng giá API là chi phí biên thực của nhà cung cấp. |

Khung gần đây của Zhu gọi token là điểm nối giữa information processing, compute, memory, energy, pricing và value; điều đáng giữ lại là cảnh báo trung tâm của nó: chi tiêu token và giá trị kinh tế là hai biến khác nhau, vì giá trị còn phụ thuộc năng suất biên, vị trí trong workflow, reasoning ẩn, rủi ro và hiệu ứng downstream. [Zhu, *AI Tokenomics* (2026)](https://arxiv.org/abs/2606.24616)

Đây cũng là lý do tôi dùng số nhiều: **tokenomics**, không phải token pricing. Giá là tín hiệu thị trường. Economics là bài toán phân bổ nguồn lực khan hiếm để tạo kết quả có giá trị.

## 1. Đừng nhầm ba con số: price, cost, value

Với một request API, chi phí mà đội sản phẩm nhìn thấy thường là

$$
C_{\text{bill}} = p_{\text{in}}T_{\text{in}} + p_{\text{out}}T_{\text{out}} + p_{\text{cache}}T_{\text{cache}} + C_{\text{tools}},
$$

trong đó $p$ là giá niêm yết theo token và $T$ là lượng token ở từng loại. Đây là **invoice**, không phải cost vật lý của provider, và càng không phải value cho người dùng.

Một accounting hữu ích hơn ở cấp workflow là:

$$
C_{\text{run}} = \sum_{k \in \text{calls}} C_{\text{bill},k}
  + C_{\text{retrieval}} + C_{\text{tool}} + C_{\text{human}} + C_{\text{failure}}.
$$

$C_{\text{failure}}$ là phần hay bị “để ngoài dashboard”: tiền hoàn, thời gian xử lý lại, một ticket bị route sai, sự mất lòng tin, hay tổn thất do một action không nên được thực hiện. Khi agent có quyền hành động, đây thường là hạng mục lớn nhất.

Cuối cùng, giá trị kỳ vọng của một run không phải “độ hay của câu trả lời”. Với một quyết định nhị phân đơn giản, hãy viết:

$$
\mathbb{E}[V] = q\,G - (1-q)\,L - C_{\text{run}},
$$

trong đó $q$ là xác suất quyết định đúng/được chấp nhận, $G$ là lợi ích nếu đúng, và $L$ là tổn thất nếu sai. Hệ quả đáng nhớ là một token tăng $q$ rất ít vẫn đáng mua trong một workflow có $L$ lớn; ngược lại, một chuỗi reasoning dài có thể vô nghĩa cho tác vụ có $G$ thấp hoặc có thể kiểm tra bằng code.

Đây là nguyên tắc **cost per accepted outcome**:

$$
\text{CPAO} = \frac{C_{\text{total over a cohort}}}{\#\,\text{outcomes accepted / successfully completed}}.
$$

CPAO không thay thế metric chất lượng, nhưng ép chúng ta gắn chi phí với kết quả. “<span>$</span>0.002/request” không nói gì nếu 30% request phải làm lại; “<span>$</span>2/case” có thể rất tốt nếu nó tránh một lỗi trị giá <span>$</span>2,000.

### Giá API đang rơi, nhưng không đồng đều

Giá đạt một mức năng lực nhất định đã giảm rất nhanh. Epoch AI, khi ghép API pricing với benchmark milestones, ước tính giá để đạt hiệu năng GPT‑4 trên GPQA Diamond đã giảm khoảng **40× mỗi năm**; giữa các benchmark/mốc năng lực, tốc độ giảm dao động **9× đến 900×/năm**. Chính Epoch cũng cảnh báo phần giảm nhanh nhất xảy ra gần đây nên chưa chắc bền vững. [Epoch AI (2025)](https://epoch.ai/data-insights/llm-inference-price-trends)

Điều này không có nghĩa “AI sẽ miễn phí”. Nó có nghĩa **biên frontier dịch chuyển**. Một preprint 2026 tìm thấy khoảng 600× giảm giá token trong dữ liệu 2020–2026, nhưng đồng thời thấy flagship/reasoning pricing không khớp một đường exponential đơn giản; tác giả diễn giải đó là *reasoning premium*. Đây là bằng chứng gợi ý, không phải định luật thị trường đã được xác nhận độc lập. [Du, *Tiered Super-Moore’s Law* (2026, preprint)](https://arxiv.org/abs/2603.28576)

Kết luận quản trị không phải là khóa một forecast giá. Đó là thiết kế sao cho economics vẫn tốt khi (a) giá giảm, (b) mix model thay đổi, và (c) người dùng tăng nhu cầu vì token rẻ hơn—hiệu ứng Jevons phiên bản AI.

## 2. Token không có “một” chi phí kỹ thuật

Transformer serving có hai pha rất khác nhau.

- **Prefill** đọc prompt và tạo KV cache. Với attention đầy đủ, công việc attention tăng gần bậc hai theo độ dài context $S$ trong pha này.
- **Decode** sinh từng token. Mỗi token mới attention vào cache dài $S$, nên chi phí mỗi bước tăng theo context đã tích lũy; tốc độ decode thường bị memory-bandwidth-bound hơn là FLOP-bound.

Một gần đúng hữu ích cho bộ nhớ KV cache là:

$$
M_{\text{KV}} \approx 2\,L\,S\,H_{\text{KV}}\,d_h\,b \quad \text{bytes},
$$

với $L$ là số layer, $S$ độ dài sequence, $H_{\text{KV}}$ số KV heads, $d_h$ head dimension, $b$ số byte mỗi phần tử, và hệ số 2 cho key và value. Chi tiết kiến trúc, quantization và sharing thay đổi hệ số, nhưng insight không đổi: **context dài là một khoản nợ bộ nhớ sống cùng request**.

Vì thế, “giá/token” không đủ để nói về serving economics. Hai workload có cùng tổng token có thể có TTFT, throughput, GPU occupancy và chi phí khác hẳn nhau nếu một workload là prompt dài + output ngắn, còn workload kia là chat nhiều lượt + decode dài.
Erdil formalizes đúng trade-off này ở cấp deployment: tại một cost/token cho trước, serial generation speed bị ràng buộc đồng thời bởi arithmetic, memory bandwidth, network bandwidth, latency, batch size và parallelism; vì vậy thực tế là một Pareto frontier speed–cost, không phải một hệ số quy đổi token cố định. [Erdil, *Inference economics of language models* (2025)](https://arxiv.org/abs/2506.04645)

PagedAttention/vLLM minh họa tại sao hệ thống có thể thay đổi economics mà không đổi model: nó quản lý KV cache như virtual memory theo block, giảm fragmentation và cho phép sharing cache; paper báo cáo throughput cao hơn **2–4×** ở cùng latency so với các hệ thống nền tảng được so sánh, rõ hơn với sequence dài và decode phức tạp. [Kwon et al., *PagedAttention* (2023)](https://arxiv.org/abs/2309.06180)

Tương tự, DistServe tách prefill và decode sang nhóm GPU khác nhau vì hai pha gây nhiễu và có ràng buộc latency khác nhau: **time to first token (TTFT)** cho prefill, **time per output token (TPOT)** cho decode. Trên các workload đánh giá, hệ thống báo cáo tới 7.4× nhiều requests hơn hoặc SLO chặt hơn 12.6×. [Zhong et al., *DistServe* (OSDI 2024)](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)

### Goodput, không phải token throughput

Throughput có thể tăng bằng batching, nhưng user không mua “tokens/second của cluster”. User mua một phản hồi kịp thời. Vì vậy metric vận hành nên là:

$$
\text{Goodput} = \#\{\text{requests hoàn thành đúng chất lượng và SLO}\}/\text{time}.
$$

Một job sinh 20,000 token mà timeout trước khi người dùng nhận được không tạo goodput. Một router giảm cost 50% nhưng làm tăng tail latency vượt SLA cũng có thể phá economics. Tối ưu thật là một bài toán frontier, không một scalar:

$$
\min \; C \quad \text{s.t.}\quad Q \ge Q^*,\; \text{TTFT}_{p95}\le \tau_1,\; \text{TPOT}_{p95}\le \tau_2,\; R\le R^*.
$$

$Q$ là quality/acceptance, $R$ là risk. Nếu chưa có các constraint này, một cost dashboard chỉ đang tối ưu thứ dễ đo.

## 3. Token ở train-time, token ở inference-time, và “số lần nó được dùng”

Chinchilla thay đổi trực giác train-time: dưới một compute budget cố định, kích thước model và số training tokens nên cùng scale; các model lớn trước đó bị undertrained. [Hoffmann et al. (2022)](https://arxiv.org/abs/2203.15556) Nhưng Chinchilla-optimal không tự động là *deployment-optimal*.

Nếu một model được suy luận $N$ lần, chi phí vòng đời tối giản là:

$$
C_{\text{lifecycle}}(\theta) = C_{\text{train}}(\theta) + N\, C_{\text{infer}}(\theta),
$$

và bài toán thực là chọn kiến trúc/training sao cho quality đạt mục tiêu với tổng chi phí nhỏ nhất. Khi $N$ lớn, một model nhỏ hơn nhưng train lâu hơn có thể thắng vì inference lặp lại rất nhiều lần. Sardana et al. mở rộng scaling laws theo hướng này và kết luận rằng với demand suy luận cỡ một tỷ requests, model nên **nhỏ hơn và train lâu hơn** Chinchilla-optimal; thí nghiệm của họ cũng thấy quality tiếp tục tăng ở vùng tokens-per-parameter cực cao. [*Beyond Chinchilla-Optimal* (2024/2025)](https://arxiv.org/abs/2401.00448)

Đây là một bài học chiến lược: model training không chỉ sản xuất “capability”; nó sản xuất một **đường chi phí biên cho mọi request tương lai**. Đó là lý do demand forecast, latency target, context distribution và cache hit rate phải có mặt sớm trong quyết định model—not appear later in a FinOps spreadsheet.

## 4. Test-time compute: khi nào một suy nghĩ nữa đáng giá?

Reasoning models làm rõ một sự thật cũ: compute có thể được dồn vào lúc inference thay vì đóng gói toàn bộ vào weights. Nhưng *nhiều thinking hơn* không phải một policy.

Gọi $b_i$ là budget token/compute dành cho prompt $i$, còn $V_i(b_i)$ là giá trị kỳ vọng sau khi đã trừ failure risk. Với ngân sách cohort $B$, bài toán lý tưởng là:

$$
\max_{b_1,\ldots,b_n} \sum_i V_i(b_i)
\quad\text{s.t.}\quad \sum_i c_i(b_i) \le B.
$$

Ở nghiệm nội suy, điều kiện kinh tế học rất đơn giản:

$$
\frac{\partial V_i/\partial b_i}{\partial c_i/\partial b_i}=\lambda.
$$

Hãy chi token tiếp cho prompt nào có **marginal value per marginal cost** cao hơn shadow price $\lambda$ của ngân sách. Prompt dễ nên dừng sớm. Prompt khó nhưng giá trị cao có thể warrant search, tool use hay verifier. Prompt khó nhưng lợi ích thấp nên escalate hoặc từ chối, không phải “think forever”.

Đây chính là động cơ sau test-time scaling có điều kiện. Snell et al. cho thấy hiệu quả của các cách scale test-time compute phụ thuộc mạnh vào độ khó prompt, và một allocation thích nghi theo prompt có thể hiệu quả hơn best-of-$N$ hơn 4× trong setting của họ. [*Scaling LLM Test-Time Compute Optimally* (2024)](https://arxiv.org/abs/2408.03314)

Vấn đề production là ta không biết trước $V_i$ và difficulty thật. Tín hiệu như entropy, self-consistency, verifier score, tool-result disagreement, hay một router score chỉ là proxy và cần calibration theo task.

### Overthinking và underthinking là cùng một lỗi phân bổ

“Cắt reasoning xuống 1,000 tokens” nghe tiết kiệm, nhưng là một fixed budget bất chấp heterogeneity. *Plan and Budget* gọi ra hai failure modes đối ngẫu: **overthinking** ở câu hỏi đơn giản và **underthinking** ở câu khó. Paper đề xuất Budget Allocation Model (BAM) và metric E3 để đánh đổi correctness với compute efficiency. [Lin et al. (2025/2026)](https://arxiv.org/abs/2505.16122)

Một stopping rule thực dụng không cần “đọc chain-of-thought”. Nó chỉ cần telemetry: dừng hoặc route sang human khi một trong các điều kiện sau đúng.

- Giá trị dự kiến của lượt compute tiếp theo thấp hơn chi phí + latency penalty.
- Verifier/grounded checks đạt ngưỡng và answer đã đủ để hành động.
- Nhiều samples/retries bất đồng, tức uncertainty không được giải bởi thêm sampling.
- Budget, quyền tool, hoặc blast radius bị chạm.

Đây là biến tokens thành policy: **một budget là một quyền chi tiêu có điều kiện**, không phải một `max_tokens` tùy ý.

## 5. Router, cascade, và cache: ba đòn bẩy kinh tế mạnh nhất

Không phải request nào cũng xứng đáng với cùng một model. Về mặt thống kê, một router nên chọn action $a \in \{\text{small}, \text{large}, \text{retrieve}, \text{human}\}$ để tối đa hóa utility điều kiện theo request $x$:

$$
a^*(x)=\arg\max_a\; \mathbb{E}[V\mid x,a]-C(a)-\rho\,R(x,a).
$$

$\rho$ là giá của risk theo use case. Một agent định giá lại giao dịch không nên dùng cùng policy với chatbot nội bộ, dù cost/token giống nhau.

**Cascade** là implementation đơn giản: thử lựa chọn rẻ trước; chỉ escalate khi confidence/verifier không đạt. FrugalGPT kết hợp prompt adaptation, approximation và cascade; trong benchmark của họ, cascade có thể match model tốt nhất với giảm chi phí tới 98%, hoặc tăng accuracy 4% ở cùng cost. Đây là kết quả thực nghiệm trong task/model của paper, không phải bảo chứng cho mọi production workload. [Chen, Zaharia & Zou (2023)](https://arxiv.org/abs/2305.05176)

RouteLLM học router từ preference data để chọn giữa model mạnh và yếu; paper báo cáo giảm chi phí hơn 2× ở một số benchmark mà không giảm chất lượng. [Ong et al. (2024/2025)](https://arxiv.org/abs/2406.18665) Một router tốt tạo *selection value*; một router không calibrated chỉ thêm một điểm hỏng và một lớp latency.

**Cache** còn sâu hơn: nếu prompt prefix lặp lại, cache không “giảm giá token” mà tránh *sản xuất lại* KV state. Nó có economic value khi workload có locality. Đo cache hit rate, prefix length saved và TTFT saved theo tenant/workflow; đừng suy ra hiệu quả chỉ từ traffic trung bình.

**Retrieval và tool calls** là factor substitution. Thay vì để model tái diễn giải 50k token, một query có provenance hoặc phép tính deterministic có thể tăng $q$ đồng thời giảm $T$. Nhưng retrieval kém có thể hạ $q$ rất nhanh. Do đó, router phải route cả **bước xử lý**, không chỉ model.

## 6. Agent tokenomics: lời hứa lớn, externality lớn

Trong single-turn chat, token chủ yếu là chi phí variable. Trong agent, token còn là message, trạng thái, điều phối và đôi khi là quyền lực. Một agent “nói chuyện với nhau” nhiều không nhất thiết làm việc tốt hơn.

Với $m$ agents, số kênh giao tiếp tiềm năng có thể tăng bậc hai:

$$
E_{\text{possible}} = \frac{m(m-1)}{2}.
$$

Không phải mọi topology chạm giới hạn đó, nhưng công thức nhắc ta rằng multi-agent có **coordination tax**: tokens để phân công, lặp context, tranh luận, kiểm tra, sửa nhau và handoff. Survey *Token Economics for LLM Agents* đặt đúng vấn đề bằng ba vai trò của token—factor sản xuất, medium trao đổi và unit of account—và dùng lăng kính micro/meso/macro để nối factor substitution, transaction cost và principal–agent problems. [Chen et al. (2026, survey)](https://arxiv.org/abs/2605.09104)

Một worker agent có thể tối ưu local reward (hoàn thành subtask, dùng nhiều evidence) nhưng làm hại global objective (trễ deadline, duplicate research, leak context). Đây là principal–agent problem theo nghĩa rất thực tế, không phải ẩn dụ kinh tế học.

### Đơn vị đúng là “trace có trách nhiệm”

Mỗi run agent nên phát ra một trace accounting tối thiểu:

| Trường | Dùng để trả lời |
| --- | --- |
| `workflow`, `task_class`, `tenant`, `policy_version` | Ai/việc gì sinh chi phí, dưới rule nào? |
| input/output/reasoning/cache token (nếu provider công bố) | Token đi đâu? |
| model, route, prompt version, retrieval/tool calls | Quyết định nào sinh token và quality? |
| TTFT, TPOT, end-to-end latency, retries | Độ trễ đến từ đâu? |
| verifier/acceptance/human intervention | Nó có tạo outcome đúng không? |
| action, reversibility, approvals, incident link | Chi phí/rủi ro downstream là gì? |

Không có trace này, “token optimization” rất dễ trở thành cắt bừa: giảm prompt nhưng mất grounding; giới hạn output nhưng tăng rework; đổi model nhưng làm yếu một cohort hiếm mà đắt.

## 7. Pricing: token budget là cơ chế screening, không chỉ là meter

Từ phía provider, token pricing giải một bài toán information asymmetry. User biết tốt hơn provider họ có task nào, cần chất lượng nào và willingness-to-pay nào. *Menu Pricing of Large Language Models* mô hình hóa user có valuation khác nhau trên continuum tasks; với technology đồng nhất, type đa chiều có thể được tóm về scalar index. Kết quả cơ chế tối ưu của paper có dạng committed-spend contracts: buyer mua budget rồi phân bổ qua token classes theo marginal cost; paper liên hệ điều này với token-budget menus, maximum/minimum spend plans, versioning nhiều model và linear API pricing. [Bergemann, Bonatti & Smolin (2025/2026)](https://arxiv.org/abs/2502.07736)

Đọc thực tế, điều đó giải thích vì sao thị trường có nhiều hình thức cùng tồn tại:

- input/output prices khác nhau;
- cached input rẻ hơn;
- batch/priority lanes;
- subscription, committed use, credits;
- model tiers và reasoning tiers;
- open-weight/self-hosting như outside option.

Không hình thức nào “tự nhiên” công bằng hơn. Mỗi hình thức chọn một cách chia surplus, chuyển rủi ro demand, và tạo incentive cho routing/caching. Khi so sánh vendor, hãy so sánh **TCO cho workload distribution**, không chỉ price card: context length, cache semantics, latency SLO, retries, egress/tools, availability, data controls, engineering cost và switching cost đều nằm trong hợp đồng kinh tế thật.

## 8. Production frontier: điểm tối ưu không phải model tốt nhất

Có thể hình dung mỗi cấu hình $j$—model, quantization, batch policy, router threshold, context policy, toolchain—là một điểm $(C_j,Q_j,L_j,R_j)$. Những điểm bị điểm khác vừa rẻ hơn, tốt hơn, nhanh hơn và an toàn hơn thì bị **dominated**. Phần còn lại là production frontier.

Một preprint gần đây xây “LLM Inference Production Frontier” trên dữ liệu WiNEval-3.0 và báo cáo diminishing marginal cost, diminishing returns to scale và một vùng cost-effectiveness tối ưu. [Zhuang et al. (2025, preprint)](https://arxiv.org/abs/2510.26136) Đóng góp quan trọng hơn con số cụ thể là cách đặt câu hỏi: benchmark accuracy đơn lẻ không thể chọn deployment.

Một framework khác gọi trade-off giữa granularity của valuation, real-time performance và allocation optimality là **Token Economics Trilemma**. Có thể đo giá trị cực chi tiết cho từng token, nhưng chính việc đo/route đó tốn latency và compute; có thể quyết định ngay, nhưng phải dùng rule thô; có thể tối ưu toàn cục, nhưng không đủ thời gian online. [Wu & Deng (2026, preprint)](https://arxiv.org/abs/2605.17410)

Tôi thấy đây là một trong những insight thực dụng nhất: một router “tối ưu” mà mất 500 ms hoặc cần full prompt để chấm điểm có thể phá chính surplus nó muốn tạo. **Chi phí quyết định cách tiêu token phải nằm trong hàm mục tiêu.**

## 9. Một operating model: tối ưu value, không tối ưu token

Đây là playbook tôi sẽ dùng để vận hành AI tokenomics trong một product team.

### Bước 1 — Viết utility contract cho từng workflow

Không bắt đầu bằng `max_tokens`. Viết: user nào, action/quyết định nào, $G$ là gì, $L$ là gì, latency nào còn hữu ích, trường hợp nào phải human-review. Đặt hard constraints cho privacy, authorization, budget và irreversibility.

### Bước 2 — Phân loại demand trước khi chọn model

Phân biệt task dễ/khó, online/offline, prompt dài/ngắn, high/low risk, repeat/novel, cacheable/non-cacheable. Average request là một hư cấu nguy hiểm. Pricing và latency đều có tail.

### Bước 3 — Lập đường cơ sở đơn giản và đo CPAO

Một model + prompt + deterministic validations thường là baseline tốt hơn một swarm agents. Đo acceptance, end-to-end latency, cost/run, CPAO và failure modes trên holdout thật.

### Bước 4 — Tách các đòn bẩy

Thử riêng từng thay đổi: context pruning, structured output, retrieval, caching, model routing, verifier, test-time budget. Nếu thay đổi năm thứ cùng lúc, bạn không biết gain đến từ đâu và không thể giữ nó khi traffic distribution đổi.

### Bước 5 — Route theo expected utility, có abstention

Cho phép “I don’t know / escalate” là một action hợp lệ. Training/evaluating router theo model preference mà không có cost-of-error sẽ route sai ở các task quan trọng nhất.

### Bước 6 — Đặt budget theo cohort và theo quyền hành động

Budget theo user, tenant, workflow, run và tool. Một agent có thể có token budget lớn cho research offline nhưng action budget rất nhỏ khi chạm hệ thống production. Budget exhaustion phải là một event observable, không phải một silent truncation.

### Bước 7 — Recompute frontier theo thời gian

Vendor đổi giá, model đổi behavior, cache locality thay đổi, demand drift. Mỗi release quan trọng phải chạy lại quality–cost–latency–risk frontier trên cùng holdout; không dùng benchmark marketing để thay thế evaluation của workload mình.

## 10. Những open problems đáng theo dõi

Không thiếu framework; thứ còn thiếu là measurement và causal evidence.

1. **Đo hidden reasoning và compute thật.** API user có thể thấy billable tokens nhưng không thấy toàn bộ compute, speculative decoding, routing nội bộ hay reasoning policy. So sánh cost-effectiveness giữa provider vì vậy thiếu transparency.
2. **Đo marginal token productivity.** Token thứ 1,000 có tăng xác suất thành công bao nhiêu cho *task distribution* thật? Cần experiments có counterfactual budgets, không chỉ correlation giữa token length và score.
3. **Calibration cho routing và stopping.** Confidence của model không phải probability đã calibration. Một router cần được đánh giá trên shift, rare/high-loss cases và thay đổi model phía sau.
4. **Value attribution trong agent traces.** Khi outcome tốt sau 12 tool calls và 4 agents, token nào, tool nào, hay human checkpoint nào tạo gain? Credit assignment quyết định tối ưu sai hay đúng.
5. **Externalities của context và multi-agent communication.** Token có thể mang PII, IP, prompt-injection payload, policy text hoặc stale facts. Chi phí privacy/security không tỷ lệ tuyến tính với price/token.
6. **SLO-aware economic scheduling.** Bài toán online phải cân fairness giữa tenants, tail latency, energy, batch efficiency và value density. “Đắt nhất trước” hay “shortest job first” đều có thể sai.
7. **Market design và interoperability.** Menu pricing, committed spend, open-weight alternatives, portability của prompts/traces và switching cost sẽ định hình ai giữ surplus khi inference commoditizes.
8. **Energy và location-aware economics.** Token price bỏ qua carbon intensity, water, grid constraint và opportunity cost của GPU. Giá rẻ có thể đẩy demand đến nơi externality cao hơn.
9. **Benchmark-to-business transfer.** Một $\Delta$ trên benchmark hiếm khi cho biết $\Delta\mathbb{E}[V]$ trong workflow. Ta cần benchmarks có acceptance, abstention, latency và cost-of-error, không chỉ exact-match.

## Kết: token rẻ không giải phóng chúng ta khỏi việc nghĩ

Khi một token rẻ đi, câu hỏi không còn là “có đủ budget để dùng AI không?” mà là “ta nên mua thêm suy nghĩ ở đâu?”

Câu trả lời tốt hiếm khi là “mọi nơi”. Nó là: mua compute ở điểm mà nó tăng xác suất một outcome có giá trị; dùng system design để không trả hai lần cho cùng context; dùng routing để không trả flagship price cho việc routine; dùng evaluation để biết suy nghĩ thêm có thực sự giúp; và dùng governance để không biến một request rẻ thành một lỗi đắt.

Đó là ý nghĩa thực tế nhất của **Tokens for Thought**. Token là đồng hồ. Công việc của chúng ta là xây một cỗ máy biết khi nào nên chạy đồng hồ, khi nào nên dừng, và làm sao mỗi nhịp đếm tạo thêm giá trị hơn rủi ro.

## Nguồn và ghi chú đọc thêm

### Kinh tế học, giá và production frontiers

- [Bergemann, Bonatti & Smolin — *Menu Pricing of Large Language Models*](https://arxiv.org/abs/2502.07736)
- [Zhu — *AI Tokenomics: The Economics of Tokens, Computation, and Pricing in Foundation Models*](https://arxiv.org/abs/2606.24616)
- [Epoch AI — *LLM inference prices have fallen rapidly but unequally across tasks*](https://epoch.ai/data-insights/llm-inference-price-trends)
- [Du — *Tiered Super-Moore’s Law*](https://arxiv.org/abs/2603.28576) *(preprint)*
- [Zhuang et al. — *Beyond Benchmarks: The Economics of AI Inference*](https://arxiv.org/abs/2510.26136) *(preprint)*
- [Wu & Deng — *Computational Challenges in Token Economics*](https://arxiv.org/abs/2605.17410) *(preprint)*

### Scaling, routing và test-time compute

- [Hoffmann et al. — *Training Compute-Optimal Large Language Models*](https://arxiv.org/abs/2203.15556)
- [Sardana et al. — *Beyond Chinchilla-Optimal*](https://arxiv.org/abs/2401.00448)
- [Snell et al. — *Scaling LLM Test-Time Compute Optimally*](https://arxiv.org/abs/2408.03314)
- [Lin et al. — *Plan and Budget*](https://arxiv.org/abs/2505.16122)
- [Chen, Zaharia & Zou — *FrugalGPT*](https://arxiv.org/abs/2305.05176)
- [Ong et al. — *RouteLLM*](https://arxiv.org/abs/2406.18665)

### Hệ thống và agents

- [Kwon et al. — *Efficient Memory Management for LLM Serving with PagedAttention*](https://arxiv.org/abs/2309.06180)
- [Zhong et al. — *DistServe* (OSDI 2024)](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)
- [Erdil — *Inference economics of language models*](https://arxiv.org/abs/2506.04645)
- [Chen et al. — *Token Economics for LLM Agents*](https://arxiv.org/abs/2605.09104) *(survey/preprint)*

Các paper đánh dấu *preprint* hữu ích để định hình câu hỏi và giả thuyết, nhưng không nên được dùng như bằng chứng độc lập duy nhất cho quyết định đầu tư. Với một production decision, hãy tái tạo trade-off trên workload, traffic và risk profile của chính mình.
