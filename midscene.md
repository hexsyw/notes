# Midscene.js

之前因为要做 mobile agent 的演示 demo，既需要实现模型和手机的交互，又希望能有一个前端页面用作输入和展示。一开始的时候，想的是和平常一样用 gradio 写个前端，后端倒是有很多可以参考的代码；试了一下发现做个页面实时显示手机画面没那么简单，想着用 scrcpy 简单有个实时给别人看就好了。后来机缘巧合，从 [ui-tars](https://github.com/bytedance/UI-TARS) 看到了 [Midscene.js](https://github.com/web-infra-dev/Midscene)，基本上满足了我的诉求。

使了几次后，发现其中有一些和我想象的设计不一样的地方。

以其介绍里面写的[例子](https://midscenejs.com/blog-support-android-automation.html#write-the-automation-scripts-by-yaml-file)为例。比如，用户的指令是 ``Connect to the device, open ebay.com, and get some items info.``，官方给了一个完成这个指令的自动化脚本示例（yaml 或者 js；yaml 如下，js 大致意思也差不多）：

```yaml
# Search headphone on eBay, extract the items info into a JSON file, and assert the shopping cart icon

android:
  deviceId: s4ey59

tasks:
  - name: search headphones
    flow:
      - aiAction: open browser and navigate to ebay.com
      - aiAction: type 'Headphones' in ebay search box, hit Enter
      - sleep: 5000
      - aiAction: scroll down the page for 800px

  - name: extract headphones info
    flow:
      - aiQuery: >
          {name: string, price: number, subTitle: string}[], return item name, price and the subTitle on the lower right corner of each item
        name: headphones

  - name: assert Filter button
    flow:
      - aiAssert: There is a Filter button on the page
```

简单来说，和我预想不一样的点在于：在 ``Midscene`` 的设计中，用户的指令并不等同于 ``aiAction``。虽然你可以把复杂的指令直接作为 aiAction 输入，但整体效果远不如拆解成数个 aiAction 形成一个脚本来得好。而在官方提供的 web playground demo 里，Action 这个概念又给人一种等同于“你就在这里输入全部指令”的感觉。

为了实现我预想的 demo 效果，今天在 AI 的帮助下，学习了一下 midscene 的代码。我想改动的部分，其实就是 aiAction 底下的逻辑，所以主要梳理了这部分的代码。

Midscene 设计了一个 ``TaskExecutor`` 来执行 ``aiAction`` 的内容；粗看下来，``TaskExecutor`` 也是 agent loop 的核心实现，在 ``action`` 中，midscene 用一个 loop 让模型分析任务、制定计划、执行计划直到任务完成或到达执行失败的边界。

```javascript
    while (true) {
      if (replanCount > replanningCycleLimit) {
        const errorMsg = `Replanning ${replanningCycleLimit} times, which is more than the limit, please split the task into multiple steps`;

        return this.appendErrorPlan(taskExecutor, errorMsg, modelConfig);
      }

      // Create planning task (automatically includes execution history if available)
      const planningTask = this.createPlanningTask(
        userPrompt,
        actionContext,
        modelConfig,
      );

      await taskExecutor.append(planningTask);
      const result = await taskExecutor.flush();
      const planResult: PlanningAIResponse = result?.output;
      if (taskExecutor.isInErrorState()) {
        return {
          output: planResult,
          executor: taskExecutor,
        };
      }

      // Execute planned actions
      const plans = planResult.actions || [];
      yamlFlow.push(...(planResult.yamlFlow || []));

      let executables: Awaited<ReturnType<typeof this.convertPlanToExecutable>>;
      try {
        executables = await this.convertPlanToExecutable(
          plans,
          modelConfig,
          cacheable,
        );
        taskExecutor.append(executables.tasks);
      } catch (error) {
        return this.appendErrorPlan(
          taskExecutor,
          `Error converting plans to executable tasks: ${error}, plans: ${JSON.stringify(
            plans,
          )}`,
          modelConfig,
        );
      }

      await taskExecutor.flush();
      if (taskExecutor.isInErrorState()) {
        return {
          output: undefined,
          executor: taskExecutor,
        };
      }

      // Check if task is complete
      if (!planResult.more_actions_needed_by_instruction) {
        break;
      }

      // Increment replan count for next iteration
      replanCount++;
    }
```

在 ``createPlanningTask`` 中，根据模型的不同调用 ``uiTarsPlanning`` 或 ``plan``，实际调用模型生成要执行的 actions。以 ``plan`` (llm-planning.ts) 为例，它提供了模型请求构造的实现。

在获取了 planning 的结果以后，通过 ``convertPlanToExecutable`` 将每个 action 转为框架定义的 task，然后进行任务的执行。
