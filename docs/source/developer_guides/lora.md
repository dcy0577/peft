<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->

# LoRA

LoRA is low-rank decomposition method to reduce the number of trainable parameters which speeds up finetuning large models and uses less memory. In PEFT, using LoRA is as easy as setting up a [`LoraConfig`] and wrapping it with [`get_peft_model`] to create a trainable [`PeftModel`].

This guide explores in more detail other options and features for using LoRA.

## Initialization

The initialization of LoRA weights is controlled by the parameter `init_lora_weights` in [`LoraConfig`]. By default, PEFT initializes LoRA weights with Kaiming-uniform for weight A and zeros for weight B resulting in an identity transform (same as the reference [implementation](https://github.com/microsoft/LoRA)).

It is also possible to pass `init_lora_weights="gaussian"`. As the name suggests, this initializes weight A with a Gaussian distribution and zeros for weight B (this is how [Diffusers](https://huggingface.co/docs/diffusers/index) initializes LoRA weights).

```py
from peft import LoraConfig

config = LoraConfig(init_lora_weights="gaussian", ...)
```

There is also an option to set `init_lora_weights=False` which is useful for debugging and testing. This should be the only time you use this option. When choosing this option, the LoRA weights are initialized such that they do *not* result in an identity transform.

```py
from peft import LoraConfig

config = LoraConfig(init_lora_weights=False, ...)
```

### LoftQ

When quantizing the base model for QLoRA training, consider using the [LoftQ initialization](https://arxiv.org/abs/2310.08659), which has been shown to improve performance when training quantized models. The idea is that the LoRA weights are initialized such that the quantization error is minimized. If you're using LoftQ, *do not* quantize the base model. You should set up a [`LoftQConfig`] instead:

```python
from peft import LoftQConfig, LoraConfig, get_peft_model

base_model = AutoModelForCausalLM.from_pretrained(...)  # don't quantize here
loftq_config = LoftQConfig(loftq_bits=4, ...)           # set 4bit quantization
lora_config = LoraConfig(..., init_lora_weights="loftq", loftq_config=loftq_config)
peft_model = get_peft_model(base_model, lora_config)
```

<Tip>

Learn more about how PEFT works with quantization in the [Quantization](quantization) guide.

</Tip>

### Rank-stabilized LoRA

Another way to initialize [`LoraConfig`] is with the [rank-stabilized LoRA (rsLoRA)](https://huggingface.co/papers/2312.03732) method. The LoRA architecture scales each adapter during every forward pass by a fixed scalar which is set at initialization and depends on the rank `r`. The scalar is given by `lora_alpha/r` in the original implementation, but rsLoRA uses `lora_alpha/math.sqrt(r)` which stabilizes the adapters and increases the performance potential from using a higher `r`.

```py
from peft import LoraConfig

config = LoraConfig(use_rslora=True, ...)
```

## Merge adapters

While LoRA is significantly smaller and faster to train, you may encounter latency issues during inference due to separately loading the base model and the LoRA adapter. To eliminate latency, use the [`~LoraModel.merge_and_unload`] function to merge the adapter weights with the base model. This allows you to use the newly merged model as a standalone model. The [`~LoraModel.merge_and_unload`] function doesn't keep the adapter weights in memory.

```py
from transformers import AutoModelForCausalLM
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-v0.1")
peft_model_id = "alignment-handbook/zephyr-7b-sft-lora"
model = PeftModel.from_pretrained(base_model, peft_model_id)
model.merge_and_unload()
```

If you need to keep a copy of the weights so you can unmerge the adapter later or delete and load different ones, you should use the [`~tuners.tuner_utils.BaseTuner.merge_adapter`] function instead. Now you have the option to use [`~LoraModel.unmerge_adapter`] to return the base model.

```py
from transformers import AutoModelForCausalLM
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-v0.1")
peft_model_id = "alignment-handbook/zephyr-7b-sft-lora"
model = PeftModel.from_pretrained(base_model, peft_model_id)
model.merge_adapter()

# unmerge the LoRA layers from the base model
model.unmerge_adapter()
```

The [`~LoraModel.add_weighted_adapter`] function is useful for merging multiple LoRAs into a new adapter based on a user provided weighting scheme in the `weights` parameter.

```py
model.add_weighted_adapter(
    adapters=["adapter_1", "adapter_2"],
    weights=[0.7, 0.3],
    adapter_name="new-weighted-adapter"
)
```

## Load adapters

Adapters can be loaded onto a pretrained model with [`~PeftModel.load_adapter`], which is useful for trying out different adapters whose weights aren't merged. Set the active adapter weights with the [`~LoraModel.set_adapter`] function.

```py
from transformers import AutoModelForCausalLM
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-v0.1")
peft_model_id = "alignment-handbook/zephyr-7b-sft-lora"
model = PeftModel.from_pretrained(base_model, peft_model_id)

# load different adapter
model.load_adapter("alignment-handbook/zephyr-7b-dpo-lora", adapter_name="dpo")

# set adapter as active
model.set_adapter("dpo")
```

To return the base model, you could use [`~LoraModel.unload`] to unload all of the LoRA modules or [`~LoraModel.delete_adapter`] to delete the adapter entirely.

```py
# unload adapter
model.unload()

# delete adapter
model.delete_adapter("dpo")
```

## Utilities

Q-LoRA paper outlines that best performance is achieved by targetting all the Linear layers of the model instead of only targetting the query and value layers. To find all the linear layers to target, we provide a utility function `get_linear_layer_names`. Below is the sample usage of it:

```py
from peft import get_linear_layer_names
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-v0.1")
print(get_linear_layer_names(model))
# ['v_proj', 'o_proj', 'k_proj', 'down_proj', 'up_proj', 'q_proj', 'gate_proj']
```