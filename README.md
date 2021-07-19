### 计算深度学习模型的吞吐量和时延测试进行部门部署和调优

神经网络的吞吐量定义为网络在单位时间内（例如，一秒）可以处理的最大输入实例数。与涉及单个实例处理的延迟不同，为了实现最大吞吐量，我们希望并行处理尽可能多的实例。有效的并行性显然依赖于数据、模型和设备。因此，为了正确测量吞吐量，我们执行以下两个步骤：（1）我们估计允许最大并行度的最佳批量大小；(2)给定这个最佳批量大小，我们测量网络在一秒钟内可以处理的实例数
要找到最佳批量大小，一个好的经验法则是达到 GPU 对给定数据类型的内存限制。这个大小当然取决于硬件类型和网络的大小。找到这个最大批量大小的最快方法是执行二进制搜索。当时间不重要时，简单的顺序搜索就足够了。为此，我们使用 for 循环将批量大小增加 1，直到达到运行时错误为止，这确定了 GPU 可以处理的最大批量大小，用于我们的神经网络模型及其处理的输入数据。
在找到最佳批量大小后，我们计算实际吞吐量。为此，我们希望处理多个批次（100 个批次就足够了），然后使用以下公式：
（批次数 X 批次大小）/（以秒为单位的总时间）
这个公式给出了我们的网络可以在一秒钟内处理的示例数量。下面的代码提供了一种执行上述计算的简单方法（给定最佳批量大小）

在bert系列模型中，max_length越长，推理时间对batch_size的增长越敏感。
模型参数量越大，推理时间对batch_size的增长越敏感。

tokenizer的速度与max_length无关，但是与batch中sentence的数量线性相关，使用berttokenizer处理长度为10的100条数据，耗时0.022左右，这时就需要根据处理速度来优化模型选择。

总结，在实际模型部署过程中，需要考虑实际场景。
一般情况下都需要考虑时延。比如限制时延的情况

区别同步调用和异步调用：
异步调用和同步调用的联系，单次处理时间不变。
区别，异步调用会减少总处理时间。

异步调用具有更高的可用性。

所以在实际部署场景下，web前后处理使用异步的方式请求tensorflow serving服务。

1. 数据量在时延时间内超出了显存占用的数据（大量数据请求）
    风控场景不需要考虑，可以控制数据量。
2. 请求量在时延时间内没有超出显存占用的数据量（可控制数量的数据请求）
    此时需要在模型的吞吐量和推理速度之间做均衡。
    具体参数调节步骤：
    1）选择合适的文本长度。
    2）选择合适的模型。
    3）选择合适的batch_size（根据此项目精细化调优），在预处理（tokenizer）和推理时间做一个均衡，单核cpu，单核gpu情况下，推理时延小于等于预处理时延（后处理忽略）。多核cpu下，预处理速度与cpu核数不是线性提升，有一定的损失（可能是内存效率）。那么要求tensorflow serving服务能够及时响应，而不会堆积太多任务。（tensorflow serving是否是并行处理，需要实验验证，官方文档描述不清，如果是，则不需考虑，如果不是，则需要考虑将模型服务重复部署到多个gpu上，提供多个接口进行服务，或者将batch_size尽量缩小）
    4）使用异步请求调用tensorflow serving服务。

在风控场景中（2B），可以控制api数据的数据条数，尽量让使用人员输送最优batch_size大小的数据。
在其他不可控的场景中，如2C的场景中，我们不能预估每时每刻有多少请求，但是我们可以设置时延，在0.01s内的请求如果少于最优batch大小，填充封装成最优batch_size，然后送入模型，如果大于最优batch大小，那么进行控制---待补充。

总之，如果有时延要求，那么我们在时延要求以及硬件设备要求的范围内，能够通过分析做到最大吞吐量。
