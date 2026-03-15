# GameplayPrediction.h 中文翻译

## 第24-264行：预测系统设计文档

```cpp
/**
 *
 *  Gameplay Ability 预测系统概述
 *
 *  高层目标：
 *  在 GameplayAbility 层面（实现一个能力时），预测是透明的。一个能力说"执行 X->Y->Z"，
 *  我们会自动预测其中可以预测的部分。我们希望避免在能力本身中编写类似
 *  "如果是权威端：执行 X。否则：执行预测版本的 X" 这样的逻辑。
 *
 *  目前，并非所有情况都已解决，但我们有一个非常坚实的客户端预测框架。
 *
 *  当我们说"客户端预测"时，我们真正指的是客户端预测游戏模拟状态。
 *  有些东西可以是"完全客户端的"，而不需要在预测系统内工作。
 *  例如，脚步声是完全客户端的，从不与此系统交互。
 *  但客户端预测施法时法力从 100 变为 90 就是"客户端预测"。
 *
 *  我们目前可以预测什么？
 *  - 初始 GameplayAbility 激活（以及带条件的链式激活）
 *  - 触发事件
 *  - GameplayEffect 应用：
 *      - 属性修改（例外：Execution 目前不预测，只有属性修改器预测）
 *      - GameplayTag 修改
 *  - Gameplay Cue 事件（来自预测性 GameplayEffect 内部或独立执行）
 *  - Montages（蒙太奇动画）
 *  - Movement（移动，内置在 UE UCharacterMovement 中）
 *
 *
 *  一些我们不预测的内容（大多数我们可能可以预测，但目前没有）：
 *  - GameplayEffect 移除
 *  - GameplayEffect 周期效果（DOT 伤害持续触发）
 *
 *
 *  我们尝试解决的问题：
 *  1. "我可以这样做吗？" - 预测的基本协议。
 *  2. "撤销" - 当预测失败时如何撤销副作用。
 *  3. "重做" - 如何避免重放我们本地预测但服务器也复制的副作用。
 *  4. "完整性" - 如何确保我们/真的/预测了所有副作用。
 *  5. "依赖关系" - 如何管理依赖预测和预测事件链。
 *  6. "覆盖" - 如何预测性地覆盖原本由服务器复制/拥有的状态。
 *
 *  ---------------------------------------------------------
 *
 *  实现细节
 *
 *  *** PredictionKey（预测键） ***
 *
 *  这个系统的一个基本概念是 Prediction Key（FPredictionKey）。
 *  预测键本身只是一个在客户端中央位置生成的唯一 ID。
 *  客户端会将预测键发送给服务器，并将预测性操作和副作用与此键关联。
 *  服务器可以对此预测键响应接受/拒绝，也会将服务器端创建的副作用与此预测键关联。
 *
 *  （重要）FPredictionKey 总是从客户端复制到服务器，但当从服务器复制到客户端时，
 *  它们*只*复制给最初将预测键发送给服务器的那个客户端。
 *  这发生在 FPredictionKey::NetSerialize 中。
 *  当客户端发送的预测键通过复制属性复制回来时，所有其他客户端将收到一个无效（0）的预测键。
 *
 *
 *  *** Ability Activation（能力激活） ***
 *
 *  Ability 激活是一等公民的预测性操作——它会生成一个初始预测键。
 *  当客户端预测性地激活一个能力时，它会显式询问服务器，服务器也会显式响应。
 *  一旦能力被预测性激活（但请求尚未发送），客户端就有一个有效的"预测窗口"，
 *  在这个窗口内可以发生预测性副作用，这些副作用不需要显式"询问"。
 *  （例如，我们不会显式询问"我可以减少法力吗，我可以让这个能力进入冷却吗"。
 *  这些操作被认为与激活能力是逻辑原子的。）
 *  你可以把这个预测窗口看作是 ActivateAbility 的初始调用栈。
 *  一旦 ActivateAbility 结束，你的预测窗口（以及你的预测键）就不再有效。
 *  这很重要，因为很多事情可以使你的预测窗口无效，比如蓝图中的任何定时器或潜在节点；
 *  我们不会跨多帧进行预测。
 *
 *
 *  AbilitySystemComponent 提供了一组函数用于客户端和服务器之间通信能力激活：
 *  TryActivateAbility -> ServerTryActivateAbility -> ClientActivateAbility(Failed/Succeed)
 *
 *  1. 客户端调用 TryActivateAbility，生成一个新的 FPredictionKey 并调用 ServerTryActivateAbility。
 *  2. 客户端继续执行（在收到服务器响应之前），调用 ActivateAbility，
 *     将生成的 PredictionKey 与 Ability 的 ActivationInfo 关联。
 *  3. 在 ActivateAbility 调用结束之前发生的任何副作用都会关联生成的 FPredictionKey。
 *  4. 服务器在 ServerTryActivateAbility 中决定能力是否真的发生，
 *     调用 ClientActivateAbility(Failed/Succeed)，
 *     并将 UAbilitySystemComponent::ReplicatedPredictionKey 设置为客户端请求中发送的生成键。
 *  5. 如果客户端收到 ClientAbilityFailed，它会立即终止能力并回滚与预测键关联的副作用。
 *      5a. "回滚"逻辑通过 FPredictionKeyDelegates 和
 *          FPredictionKey::NewRejectedDelegate/NewCaughtUpDelegate/NewRejectOrCaughtUpDelegate 注册。
 *      5b. ClientAbilityFailed 实际上是我们"拒绝"预测键的唯一情况，
 *          因此我们所有的当前预测都依赖于能力是否激活。
 *  6. 如果 ServerTryActivateAbility 成功，客户端必须等待属性复制追上
 *     （Succeed RPC 会立即发送，属性复制会在其自己的时机发生）。
 *     一旦 ReplicatedPredictionKey 追上了之前步骤使用的键，客户端就可以撤销其预测性副作用。
 *     参见 FReplicatedPredictionKeyItem::OnRep 了解 CatchUpTo 逻辑。
 *     参见 UAbilitySystemComponent::ReplicatedPredictionKeyMap 了解键如何实际复制。
 *     参见 ~FScopedPredictionWindow 了解服务器如何确认键。
 *
 *
 *  *** GameplayEffect Prediction（GameplayEffect 预测） ***
 *
 *  GameplayEffects 被视为能力激活的副作用，不会单独被接受/拒绝。
 *
 *  1. GameplayEffects 只有在有有效预测键时才会在客户端应用。
 *     （如果没有预测键，客户端会跳过应用）。
 *  2. 如果 GameplayEffect 被预测，属性、GameplayCues 和 GameplayTags 都会被预测。
 *  3. 当创建 FActiveGameplayEffect 时，它存储预测键（FActiveGameplayEffect::PredictionKey）
 *      3a. 即时效果在下面的"属性预测"中解释。
 *  4. 在服务器上，相同的预测键也会设置在将要复制下去的服务器 FActiveGameplayEffect 上。
 *  5. 作为客户端，如果你收到一个带有有效预测键的复制 FActiveGameplayEffect，
 *     你检查是否有具有相同键的 ActiveGameplayEffect，如果有匹配，
 *     我们不应用"应用时"类型的逻辑，例如 GameplayCues。这解决了"重做"问题。
 *     但我们的 ActiveGameplayEffects 容器中暂时会有两个"相同"的 GameplayEffect：
 *  6. 同时，FReplicatedPredictionKeyItem::OnRep 会追上，预测性效果会被移除。
 *     在这种情况下移除它们时，我们再次检查 PredictionKey 并决定是否应该不执行"移除时"逻辑/GameplayCue。
 *
 *  此时，我们已经有效地预测了一个 GameplayEffect 作为副作用，并处理了"撤销"和"重做"问题。
 *
 *  参见 FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec，
 *  它注册了追上时要做什么（RemoveActiveGameplayEffect_NoReturn）。
 *  参见 FActiveGameplayEffect::PostReplicatedAdd、FActiveGameplayEffect::PreReplicatedRemove
 *  和 FGameplayCue::PostReplicatedAdd，了解 FPredictionKey 如何与 GE 和 GC 关联的示例。
 *
 *
 *  *** Attribute Prediction（属性预测） ***
 *
 *  由于属性作为标准 UProperty 复制，预测对它们的修改可能很棘手（"覆盖"问题）。
 *  瞬时修改可能更难，因为这些本质上是无状态的。
 *  （例如，如果修改后没有记录，回滚属性修改是很困难的）。
 *  这使得这种情况下的"撤销"和"重做"问题也很困难。
 *
 *  基本的攻击计划是将属性预测视为增量预测而非绝对值预测。
 *  我们不预测我们有 90 法力，我们预测我们相对于服务器值有 -10 法力，
 *  直到服务器确认我们的预测键。
 *  基本上，将即时修改视为/无限持续修改/来处理属性，当它们被预测性执行时。
 *  这解决了"撤销"和"重做"问题。
 *
 *  对于"覆盖"问题，我们可以在属性的 OnRep 中处理，
 *  将复制的（服务器）值视为属性的"基础值"而非"最终值"，
 *  并在复制发生后重新聚合我们的"最终值"。
 *
 *
 *  1. 我们将预测性即时 GameplayEffect 视为无限持续 GameplayEffect。
 *     参见 UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf。
 *  2. 我们必须*始终*在属性上收到 RepNotify 调用
 *     （不仅仅是从上次本地值发生变化时，因为我们会提前预测变化）。
 *     使用 REPNOTIFY_Always 完成。
 *  3. 在属性 RepNotify 中，我们调用 AbilitySystemComponent::ActiveGameplayEffects
 *     来更新我们的"最终值"，给定新的"基础值"。GAMEPLAYATTRIBUTE_REPNOTIFY 可以做到这一点。
 *  4. 其他一切都像上面一样工作（GameplayEffect 预测）：
 *     当预测键追上时，预测性 GameplayEffect 被移除，我们将返回服务器给定的值。
 *
 *
 *  示例：
 *
 *  void UMyHealthSet::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
 *  {
 *      Super::GetLifetimeReplicatedProps(OutLifetimeProps);
 *
 *      DOREPLIFETIME_CONDITION_NOTIFY(UMyHealthSet, Health, COND_None, REPNOTIFY_Always);
 *  }
 *
 *  void UMyHealthSet::OnRep_Health()
 *  {
 *      GAMEPLAYATTRIBUTE_REPNOTIFY(UMyHealthSet, Health);
 *  }
 *
 *
 *  *** Gameplay Cue Events（Gameplay Cue 事件） ***
 *
 *  除了已经解释的 GameplayEffects 之外，Gameplay Cues 可以独立激活。
 *  这些函数（UAbilitySystemComponent::ExecuteGameplayCue 等）考虑了网络角色和预测键。
 *
 *  1. 在 UAbilitySystemComponent::ExecuteGameplayCue 中，如果是权威端则执行多播事件（带复制键）。
 *     如果非权威端但有有效预测键，预测 GameplayCue。
 *  2. 在接收端（NetMulticast_InvokeGameplayCueExecuted 等），如果有复制键，则不执行事件（假设你已预测）。
 *
 *  记住 FPredictionKeys 只复制给发起的所有者。这是 FReplicationKey 的固有属性。
 *
 *
 *  *** Triggered Data Prediction（触发数据预测） ***
 *
 *  触发数据目前用于激活能力。本质上这都通过 ActivateAbility 的相同代码路径。
 *  不是从输入按下激活能力，而是从另一个游戏代码驱动的事件激活。
 *  客户端能够预测性执行这些事件，从而预测性激活能力。
 *
 *  但有一些细微差别，因为服务器也会运行触发事件的代码。
 *  服务器不会只是等待从客户端听到消息。
 *  服务器会保留一个从预测性能力激活的触发能力列表。
 *  当收到来自触发能力的 TryActivate 时，服务器会查看它是否已经运行了这个能力，
 *  并用该信息响应。
 *
 *  问题是我们没有正确回滚这些操作。触发事件和复制还有待完成的工作。（在最后解释）。
 *
 *
 *  ---------------------------------------------------------
 *
 *  高级话题！
 *
 *  *** Dependencies（依赖关系） ***
 *
 *  我们可能有这样的情况："能力 X 激活并立即触发一个事件，该事件激活能力 Y，
 *  Y 又触发另一个能力 Z"。依赖链是 X->Y->Z。
 *  这些能力中的每一个都可能被服务器拒绝。如果 Y 被拒绝，那么 Z 也从未发生，
 *  但服务器从未尝试运行 Z，所以服务器不会显式决定"不，Z 不能运行"。
 *
 *  为了处理这个问题，我们有一个 Base PredictionKey 的概念，它是 FPredictionKey 的成员。
 *  当调用 TryActivateAbility 时，我们传入当前的 PredictionKey（如果适用）。
 *  该预测键用作生成的任何新预测键的基础。
 *  我们以这种方式建立键链，然后可以在 Y 被拒绝时使 Z 无效。
 *
 *  但这稍微有些微妙。在 X->Y->Z 的情况下，服务器在尝试运行链本身之前
 *  只会收到 X 的 PredictionKey。
 *  例如，它会用从客户端发送的原始预测键 TryActivate Y 和 Z，
 *  而客户端每次调用 TryActivateAbility 时都会生成新的 PredictionKey。
 *  客户端*必须*为每次能力激活生成新的 PredictionKey，因为每次激活不是逻辑原子的。
 *  事件链中产生的每个副作用必须有唯一的 PredictionKey。
 *  我们不能让 X 中产生的 GameplayEffects 与 Z 中产生的有相同的 PredictionKey。
 *
 *  为了解决这个问题，X 的预测键被视为 Y 和 Z 的 Base 键。
 *  Y 到 Z 的依赖完全保留在客户端，通过 FPredictionKeyDelegates::AddDependancy 完成。
 *  我们添加委托，如果 Y 被拒绝/确认，则拒绝/追上 Z。
 *
 *  这个依赖系统允许我们在单个预测窗口/范围内有多个非逻辑原子的预测性操作。
 *
 *  但有一个问题：因为依赖保留在客户端，服务器实际上不知道它之前是否拒绝了依赖操作。
 *  你可以通过在 Gameplay Ability 中使用激活标签来设计解决这个问题。
 *  例如，当预测依赖 GA_Combo1 -> GA_Combo2 时，
 *  你可以让 GA_Combo2 只在有 GA_Combo1 给予的 GameplayTag 时才激活。
 *  因此，GA_Combo1 的拒绝也会导致服务器拒绝 GA_Combo2 的激活。
 *
 *
 *  *** Additional Prediction Windows（额外的预测窗口，在 Ability 内） ***
 *
 *  如前所述，预测键只在单个逻辑范围内可用。
 *  一旦 ActivateAbility 返回，我们基本上完成了该键的使用。
 *  如果能力正在等待外部事件或定时器，在我们准备继续执行时，
 *  可能已经收到了服务器的确认/拒绝。
 *  因此，初始激活后产生的任何额外副作用不能再绑定到原始键的生命周期。
 *
 *  这并不太糟糕，除了能力有时会对玩家输入做出反应。
 *  例如，一个"按住蓄力"能力想要在按钮释放时立即预测一些东西。
 *  可以使用 FScopedPredictionWindow 在能力内创建新的预测窗口。
 *
 *  FScopedPredictionWindow 提供了一种向服务器发送新预测键的方法，
 *  并让服务器在相同的逻辑范围内获取和使用该键。
 *
 *  UAbilityTask_WaitInputRelease::OnReleaseCallback 是一个很好的例子。
 *  事件流程如下：
 *  1. 客户端进入 UAbilityTask_WaitInputRelease::OnReleaseCallback 并启动新的 FScopedPredictionWindow。
 *     这为此作用域创建新的预测键（FScopedPredictionWindow::ScopedPredictionKey）。
 *  2. 客户端调用 AbilitySystemComponent->ServerInputRelease，
 *     将 ScopedPrediction.ScopedPredictionKey 作为参数传递。
 *  3. 服务器运行 ServerInputRelease_Implementation，
 *     获取传入的 PredictionKey 并用 FScopedPredictionWindow 将其设置为
 *     UAbilitySystemComponent::ScopedPredictionKey。
 *  4. 服务器在/相同的作用域内/运行 UAbilityTask_WaitInputRelease::OnReleaseCallback。
 *  5. 当服务器在 ::OnReleaseCallback 中遇到 FScopedPredictionWindow 时，
 *     它从 UAbilitySystemComponent::ScopedPredictionKey 获取预测键。
 *     现在该键用于此逻辑范围内的所有副作用。
 *  6. 一旦服务器结束此作用域预测窗口，使用的预测键就完成并设置为 ReplicatedPredictionKey。
 *  7. 此作用域内创建的所有副作用现在在客户端和服务器之间共享一个键。
 *
 *  这工作的关键是 ::OnReleaseCallback 调用 ::ServerInputRelease，
 *  后者在服务器上调用 ::OnReleaseCallback。
 *  没有空间让其他任何事情发生并使用给定的预测键。
 *
 *  虽然在这个例子中没有"Try/Failed/Succeed"调用，
 *  但所有副作用都是程序化分组/原子的。
 *  这解决了在服务器和客户端上运行的任意函数调用的"撤销"和"重做"问题。
 *
 *
 *  ---------------------------------------------------------
 *
 *  不支持 / 问题 / 待办
 *
 *  触发事件不会显式复制。例如，如果触发事件只在服务器上运行，
 *  客户端永远不会知道。这也阻止了我们做跨玩家/AI 等事件。
 *  最终应该添加对此的支持，它应该遵循 GameplayEffect 和 GameplayCues 遵循的相同模式
 *  （用预测键预测触发事件，如果它有预测键则忽略 RPC 事件）。
 *
 *  这个整个系统的一个大警告：任何链式激活（包括触发事件）的回滚目前开箱即用是不可能的。
 *  原因是每个 ServerTryActivateAbility 会按顺序响应。
 *  让我们以链式依赖 GA 为例：GA_Mispredict -> GA_Predict1。
 *  在这个例子中，当 GA_Mispredict 被激活并在本地预测时，
 *  它也会立即激活 GA_Predict1。
 *  客户端为 GA_Mispredict 发送 ServerTryActivateAbility，
 *  服务器拒绝它（发送回 ClientActivateAbilityFailed）。
 *  目前，我们没有任何委托在客户端上拒绝依赖能力（服务器甚至不知道有依赖关系）。
 *  在服务器上，它也收到 GA_Predict1 的 ServerTryActivateAbility。
 *  假设那成功了，客户端和服务器现在都在执行 GA_Predict1，
 *  尽管 GA_Mispredict 从未发生。
 *  你可以通过使用标签系统确保 GA_Mispredict 成功来设计解决这个问题。
 *
 *
 *  *** 预测"Meta"属性如 Damage/Healing 与"真实"属性如 Health ***
 *
 *  我们无法预测性地应用 Meta 属性。
 *  Meta 属性只在即时效果上工作，在 GameplayEffect 的后端
 *  （UAttributeSet 上的 Pre/Post Modify Attribute）。
 *  当应用持续时间的 GameplayEffect 时，这些事件不会被调用。
 *  例如，一个修改伤害持续 5 秒的 GameplayEffect 没有意义。
 *
 *  为了支持这一点，我们可能会添加一些对持续 Meta 属性的有限支持，
 *  并将即时 GameplayEffect 的转换从前端
 *  （UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf）
 *  移到后端（UAttributeSet::PostModifyAttribute）。
 *
 *
 *  *** 预测持续的乘法 GameplayEffects ***
 *
 *  预测基于百分比的游戏效果时也有限制。
 *  由于服务器复制属性的"最终值"，而不是修改它的整个聚合器链，
 *  我们可能会遇到客户端无法准确预测新 GameplayEffect 的情况。
 *
 *  例如：
 *  - 客户端有一个永久 +10% 移动速度 buff，基础移动速度 500 -> 550 是此客户端的最终移动速度。
 *  - 客户端有一个能力授予额外的 10% 移动速度 buff。
 *    预期是将基于百分比的乘法器*求和*，最终 20% 加成到 500 -> 600 移动速度。
 *  - 然而在客户端，我们只是将 10% buff 应用到 550 -> 605。
 *
 *  这需要通过复制属性的聚合器链来修复。
 *  我们已经复制了一些这样的数据，但不是完整的修改器列表。
 *  我们最终需要研究支持这个。
 *
 *
 *  *** "Weak Prediction"（弱预测） ***
 *
 *  我们可能仍然会有不太适合这个系统的情况。
 *  有些情况下预测键交换是不可行的。
 *  例如，一个能力，玩家碰撞/接触的任何人都收到一个减慢他们的 GameplayEffect
 *  并使他们的材质变蓝。
 *  由于我们不能每次发生这种情况时都发送服务器 RPC
 *  （服务器也不一定能在其模拟点处理消息），
 *  没有办法在客户端和服务器之间关联 GameplayEffect 副作用。
 *
 *  这里的一种方法可能是考虑一种较弱形式的预测。
 *  一种不使用新鲜预测键，而是服务器假设客户端会预测整个能力的所有副作用的预测。
 *  这至少会解决"重做"问题，但不会解决"完整性"问题。
 *  如果客户端预测可以尽可能最小化——例如只预测初始粒子效果
 *  而不是预测状态和属性变化——那么问题就不那么严重。
 *
 *  我可以设想一种弱预测模式，这是（某些能力？所有能力？）
 *  在没有新鲜预测键可以准确关联副作用时回退到的模式。
 *  在弱预测模式下，也许只有某些操作可以被预测——
 *  例如 GameplayCue 执行事件，但不是 OnAdded/OnRemove 事件。
 *
 *
 */


/**
 *  FPredictionKey 是支持 GameplayAbility 系统中客户端预测的通用方式。
 *  FPredictionKey 本质上是一个 ID，用于标识在客户端上完成的预测性操作和副作用。
 *  UAbilitySystemComponent 支持预测键及其副作用在客户端和服务器之间的同步。
 *
 *  本质上，任何东西都可以与 PredictionKey 关联，例如激活一个 Ability。
 *  客户端可以生成一个新鲜的 PredictionKey 并在其 ServerTryActivateAbility 调用中发送给服务器。
 *  服务器可以确认或拒绝此调用（ClientActivateAbilitySucceed/Failed）。
 *
 *  当客户端预测其能力时，它正在创建副作用（GameplayEffects、TriggeredEvents、Animations 等）。
 *  当客户端预测这些副作用时，它将每个副作用与能力激活开始时生成的预测键关联。
 *
 *  如果能力激活被拒绝，客户端可以立即恢复这些副作用。
 *  如果能力激活被接受，客户端必须等待复制的副作用发送到服务器。
 *      （ClientActivatbleAbilitySucceed RPC 会立即发送。
 *       属性复制可能在几帧后发生。）
 *      一旦服务器创建的副作用复制完成，客户端就可以撤销其本地预测的副作用。
 *
 *  FPredictionKey 本身提供的主要功能是：
 *      - 唯一 ID 和一个拥有依赖预测键链的系统（"Current"和"Base"整数）
 *      - 一个特殊的 ::NetSerialize 实现 *** 只将预测键序列化给预测客户端 ***
 *          - 这很重要，因为它允许我们在复制状态中序列化预测键，
 *            知道只有给服务器预测键的客户端才会真正看到它们！
 *
 */
```
