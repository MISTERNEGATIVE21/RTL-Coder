```
  ___    _____   _        ____               _               
 |  _ \  |_   _| | |      / ___|   ___     __| |   ___   _ __ 
 | |_) |   | |   | |     | |      / _ \   / _` |  / _ \ | '__|
 |  _ <    | |   | |___  | |___  | (_) | | (_| | |  __/ | |   
 |_| \_\   |_|   |_____|  \____|  \___/   \__,_|  \___| |_|                                           
```
# RTL-Coder: Outperforming GPT-3.5 in RTL Code Generation with Our Fully Open-Source Dataset and Lightweight Solution
Shang Liu, Wenji Fang, Yao Lu, Qijun Zhang, Hongce Zhang, and Zhiyao Xie, "RTL-Coder: Outperforming GPT-3.5 in RTL Code Generation with Our Fully Open-Source Dataset and Lightweight Solution"[[paper]](https://arxiv.org/abs/2312.08617)

_**Note**: The model, dataset, inference scripts, data generation flow and training flow are provided._

Targeting Verilog code generation, we propose an automated flow to generate a large labeled dataset with over 27,000 diverse Verilog design problems and answers. It addresses the serious data availability challenge in IC design-related tasks, and its potential applications are not limited to LLMs. The
LLM directly trained on it can already achieve comparable accuracy with GPT-3.5.

We also introduce a new LLM training scheme based on code quality feedback. It further boosts the ultimate model performance to outperform GPT-3.5. And we further revised the training process from an algorithm perspective to reduce its GPU memory consumption.




TABLE 1 summarizes existing works in LLM-based design RTL generation.

<img src="_pic/LLM4RTL_comparison_arxiv.jpg" width="500px">

TABLE 1: LLM-based works on design RTL generation (e.g., Verilog). 

**In our work, we provide four RTL code generation models that are available on the HuggingFace platform.**

1. [RTLCoder-Deepseek-v1.1](https://huggingface.co/ishorn5/RTLCoder-Deepseek-v1.1).
   This model was finetund on DeepSeek-coder-6.7b. It has the best performance on VerilogEval and RTLLM benchmarks but with a relatively lower inference speed compared with the following models. The       RTLCoder-Deepseek-v1.1 may not stop even when the required output text is finished. So We need to extract the required code part before the keyword"endmodulemodule" from the output sequence and add an "endmodule" at the end.
3. [RTLCoder-v1.1](https://huggingface.co/ishorn5/RTLCoder-v1.1). (Finetuned based on Mistral-v0.1)
4. [RTLCoder-v1.1-gptq-4bit](https://huggingface.co/ishorn5/RTLCoder-v1.1-gptq-4bit). (The GPTQ version of RTLCoder-v1.1)
5. [RTLCoder-v1.1-gguf-4bit](https://huggingface.co/ishorn5/RTLCoder-v1.1-gguf-4bit). This quantized one could run on CPU. (The CPU version of RTLCoder-v1.1)

## 1. Working flow overview
In this paper, there are two main contributions to obtain the RTLCoder. 
(1) We first introduce our automated dataset generation flow. It generated our RTL generation dataset with over 27 thousand samples, each sample being a pair of design description instruction and corresponding reference code. We build this automated generation flow by taking full advantage
of the powerful general text generation ability of the commercial tool GPT. Please notice that GPT is only used for dataset generation in this work and we adhere to the terms of service of OpenAI, and there is no commercial competition between the proposed RTLcoder and OpenAI's models. The automated dataset generation flow is illustrated in **Figure 1** which includes three stages: 1) RTL domain keywords preparation, 2) instruction generation, and 3) reference code generation. We designed several general prompt templates to control GPT generating the desired outputs in each stage.


   <img src="_pic/data_gen_flow.jpg" width="700px">

   Figure 1:  Our proposed automated dataset generation flow.

(2) Besides the new training dataset, we propose a new LLM training scheme that incorporates code quality scoring. It significantly improves the RTLCoder’s performance on the RTL generation task. Also, we revised the training process from the algorithm perspective to reduce the GPU memory consumption of this new training method, allowing implementation with limited hardware resources. The training scheme is illustrated in **Figure 2**.


   <img src="_pic/training_flow.jpg" width="700px">

   Figure 2:  Our proposed training scheme based on RTL quality score.


## 2. Training data generation
We provide the generation scripts and data samples in the folder **"data_generation"**. You can design your own prompting method by modifying the file **"p_example.txt"** and **"instruction_gen.py"**.

You can expand the existing dataset by running the following command.
```
python instruction_gen.py
```
The 27K instruction-code dataset "Resyn-27k.json" is provided in the "dataset" file. Please kindly note that the dataset was generated by GPT-3.5-turbo and it cannot be guaranteed that all the data are strictly correct. Despite the possible presence of errors in some problem descriptions and design code, we believe that they can still provide valuable information for model training.

## 3. Model inference

(1) Inference demo

The input prompt may have a great influence on the generation quality. Ideally, it should describe the circuit "IO" and behavior clearly so that it doesn't contain ambiguity. We provide a template as follows.
```
Please act as a professional verilog designer.

Implement a traffic light, with red, yellow and green three small indicators and a pedestrian button, under normal circumstances, the motor vehicle lane indicator light according to 60 clock cycles of green, 5 clock cycles of yellow, 10 clock cycles of red. When the pedestrian button is pressed, if the remaining green time is greater than 10 clocks, it is shortened to 10 clocks, and if it is less than 10 clocks, it remains unchanged. The lane light and the sidewalk light should be paired, when the lane light is green or yellow, the sidewalk light is red; When the lane light is red, the sidewalk light is green, and for the sake of simplicity, only the lane light is considered.

Module name:  
    traffic_light

Inputs:
    rst_n: Reset signal (active low).
    clk: Clock signal.
    pass_request: Request signal for allowing vehicles to pass.

Outputs:
    clock[7:0]: An 8-bit output representing the count value of the internal counter.
    red, yellow, green: Output signals representing the state of the traffic lights.

Parameters:
    idle, s1_red, s2_yellow, s3_green: Enumeration values representing different states of the traffic light controller.

Registers and Wires:
    cnt: A 8-bit register used as an internal counter for timing purposes.
    state: A 2-bit register representing the current state of the traffic light controller.
    p_red, p_yellow, p_green: 1-bit registers representing the next values for the red, yellow, and green signals.

Implementation:
The following is the design track we recommend:
The first always block is responsible for the state transition logic. It uses a case statement to handle different states. Here's a summary of each state:
idle: Initial state where all signals are set to 0. Transition to s1_red state occurs immediately.
s1_red: Sets the red signal to 1 and waits for a count of 3 before transitioning to s3_green state. Otherwise, it remains in s1_red state.
s2_yellow: Sets the yellow signal to 1 and waits for a count of 3 before transitioning to s1_red state. Otherwise, it remains in s2_yellow state.
s3_green: Sets the green signal to 1 and waits for a count of 3 before transitioning to s2_yellow state. Otherwise, it remains in s3_green state.
The second always block handles the counting logic of the internal counter (cnt). The counter is decremented by 1 on every positive edge of the clock or negative edge of the reset signal. The counter values are adjusted based on various conditions:
If (!rst_n), the counter is set to 10.
If the pass_request signal is active and the green signal is active, the counter is set to 10.
If the green signal is inactive and the previous green signal (p_green) was active, the counter is set to 60.
If the yellow signal is inactive and the previous yellow signal (p_yellow) was active, the counter is set to 5.
If the red signal is inactive and the previous red signal (p_red) was active, the counter is set to 10.
Otherwise, the counter is decremented normally.
The assign statement assigns the value of the internal counter (cnt) to the output clock.
The final always block handles the output signals. It assigns the previous values (p_red, p_yellow, p_green) to the output signals (red, yellow, green) on the positive edge of the clock or negative edge of the reset signal.

Give me the complete code.
module traffic_light
    (
		input rst_n, 
      input clk, 
      input pass_request,
		  output wire[7:0]clock,
      output reg red,
		  output reg yellow,
		  output reg green
    );
	
	parameter 	idle = 2'd0,
				s1_red = 2'd1,
				s2_yellow = 2'd2,
				s3_green = 2'd3;
```

If you don't have a GPU with more than 4 GB memory, please try the quantized 4-bit version which could run on CPU: [RTLCoder-v1.1-gguf-4bit](https://huggingface.co/ishorn5/RTLCoder-v1.1-gguf-4bit).

```

from ctransformers import AutoModelForCausalLM
model_path = 'ggml-model-q4_0.gguf'
# Set gpu_layers to the number of layers to offload to GPU. Set to 0 if no GPU acceleration is available on your system.
llm = AutoModelForCausalLM.from_pretrained(model_path, model_type="mistral", gpu_layers=0, max_new_tokens=2000, context_length=6048, temperature=0.5, top_p=0.95,)
prompt = "Please act as a professional verilog designer and provide a half adder. \nmodule half_adder\n(input a, \ninput b, \noutput sum, \n output carry);\n"
print(llm(prompt))

```
For inference using RTLCoder, you can just use the following code.
```
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM
# Prompt
prompt = "Please act as a professional verilog designer and provide a half adder. \nmodule half_adder\n(input a, \ninput b, \noutput sum, \n output carry);\n"

# Load model and tokenizer
# With multiple gpus, you can specify the GPU you want to use as gpu_name (e.g. int(0)).
gpu_name = 0
tokenizer = AutoTokenizer.from_pretrained("ishorn5/RTLCoder-Deepseek-v1.1")
model = AutoModelForCausalLM.from_pretrained("ishorn5/RTLCoder-Deepseek-v1.1", torch_dtype=torch.float16, device_map=gpu_name)
model.eval()
# Sample
input_ids = tokenizer(prompt, return_tensors="pt").input_ids.to(gpu_name)
sample = model.generate(input_ids, max_length=512, temperature=0.5, top_p=0.9)
s_full = tokenizer.decode(sample[0])
# The RTLCoder-Deepseek-v1.1 may not stop even when the required output text is finished.
# We need to extract the required part from the output sequence based on a keyword "endmodulemodule".
s = s_full.split('endmodulemodule', 1)[0] + "\n" + "endmodule" 
print(s)

#For "ishorn5/RTLCoder-v1.1", it will stop generating tokens after completing the coding task.
#But you can still use the keyword "endmodule" to extract the code part.
#tokenizer = AutoTokenizer.from_pretrained("ishorn5/RTLCoder-v1.1")
#model = AutoModelForCausalLM.from_pretrained("ishorn5/RTLCoder-v1.1", torch_dtype=torch.float16, device_map=gpu_name)
#model.eval()
#Sample
#input_ids = tokenizer(prompt, return_tensors="pt").input_ids.to(gpu_name)
#sample = model.generate(input_ids, max_length=512, temperature=0.5, top_p=0.9)
#print(tokenizer.decode(sample[0]))
```
To test the RTLCoder-gptq-4bit,  you can just use the following code.
```
from transformers import AutoTokenizer
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig
# Prompt
prompt = "Please act as a professional verilog designer and provide a half adder. \nmodule half_adder\n(input a, \ninput b, \noutput sum, \n output carry);\n"

tokenizer = AutoTokenizer.from_pretrained("ishorn5/RTLCoder-v1.1-gptq-4bit", use_fast=True)
model = AutoGPTQForCausalLM.from_quantized("ishorn5/RTLCoder-v1.1-gptq-4bit", device="cuda:0")
model.eval()
# Sample
inputs = tokenizer(prompt, return_tensors="pt").to(0)
sample = model.generate(**inputs, max_length=512, temperature=0.5, top_p=0.9)
print(tokenizer.decode(sample[0]))
```

(2) Test model on Verilog-eval

We provide the inference script **"test_on_verilog-eval.py"** for "verilog-eval" benchmark in folder **"benchmark_inference"**. 
You need to firstly download the "verilog-eval" benchmark.
```
git clone https://github.com/NVlabs/verilog-eval.git
```
Then modify the **"descri_path"** and **"input_path"** in **"test_on_nvbench.py"** according to the location of verlog-eval file.  

Use the following command to do model inference on EvalMachine.
```
python test_on_nvbench.py --model <your model path or "ishorn5/RTLCoder-v1.1"> --n 20 --temperature=0.2 --gpu_name 0 --output_dir <your result directory> --output_file <your result file, e.g. rtlcoder_temp0.2_evalmachine.json> --bench_type Machine
```
If you want to do model inference on EvalHuman, just change the --bench_type from Machine to Human.
```
python test_on_nvbench.py --model <your model path or "ishorn5/RTLCoder-v1.1"> --n 20 --temperature=0.2 --gpu_name 0 --output_dir <your result directory> --output_file <your result file, e.g. rtlcoder_temp0.2_evalhuman.json> --bench_type Human
```
Please refer the verilog-eval benchmark repo https://github.com/NVlabs/verilog-eval to evaluate the generated code quality.

(3) Test model on RTLLM

We provide the organized  descriptions of RTLLM as **"rtllm-1.1.json"**  in folder **"benchmark_inference"**. 

Use the following command to do inference on RTLLM benchmark.
```
python test_on_rtllm.py --model <your model path or "ishorn5/RTLCoder-v1.1">  --n 5 --temperature=0.5 --gpu_name 0 --output_dir <your result directory> 
```
Please refer the RTLLM benchmark repo https://github.com/hkust-zhiyao/RTLLM to evaluate the generated code quality.

## 4. Model training
We provide three options for instruction tuning: MLE based direct train, Scoring train and Scoring train with gradients splitting. For more details, please refer to the paper and the folder **"train"**.

For MLE based direct training, just simply use:
```
torchrun --nproc_per_node=4  mle.py \
    --model_name_or_path <model path> \
    --data_path <data path> \
    --fp16 True \
    --output_dir <output path>\
    --num_train_epochs 3 \
    --per_device_train_batch_size 2 \
    --per_device_eval_batch_size 2 \
    --gradient_accumulation_steps 32 \
    --evaluation_strategy "no" \
    --save_strategy "steps" \
    --save_steps 50 \
    --save_total_limit 10 \
    --learning_rate 1e-5 \
    --weight_decay 0. \
    --logging_steps 1 \
    --tf32 False\
    --gradient_checkpointing True \
    --deepspeed ds_stage_2.json\
    --model_max_length 2048
```
For scoring based training method, you need to firstly obtain answer candidates to each of the instruction in the training dataset and we provide a data sample **"scoring_data_sample.json"** to illustrate the  data format for training.
Then use the following command.

```
torchrun --nproc_per_node=4  mle_scoring.py \
    --model_name_or_path <model path> \
    --data_path <data path> \
    --fp16 True \
    --output_dir <output path>\
    --num_train_epochs 3 \
    --per_device_train_batch_size 1 \
    --per_device_eval_batch_size 2 \
    --gradient_accumulation_steps 64\
    --evaluation_strategy "no" \
    --save_strategy "steps" \
    --save_steps 50 \
    --save_total_limit 10 \
    --learning_rate 1e-5 \
    --weight_decay 0. \
    --logging_steps 1 \
    --tf32 False\
    --gradient_checkpointing True \
    --deepspeed ds_stage_2.json\
    --model_max_length 2048
```

If your gpu could't afford batch size 1 with these answer candidates, try the gradients splitting method.
```
torchrun --nproc_per_node=4  mle_scoring_grad_split.py \
    --model_name_or_path <model path> \
    --data_path <data path> \
    --fp16 True \
    --output_dir <output path>\
    --num_train_epochs 3 \
    --per_device_train_batch_size 1 \
    --per_device_eval_batch_size 2 \
    --gradient_accumulation_steps 64\
    --evaluation_strategy "no" \
    --save_strategy "steps" \
    --save_steps 50 \
    --save_total_limit 10 \
    --learning_rate 1e-5 \
    --weight_decay 0. \
    --logging_steps 1 \
    --tf32 False\
    --gradient_checkpointing True \
    --deepspeed ds_stage_2.json\
    --model_max_length 2048
```

