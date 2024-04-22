# GEN AI - Generative AI Introduction

## Common terminology
1. **Models:**  
They go through large datasets in process called **Training** and find patterns and make predictions accordingly.
   They **DO NOT** store that information, so they cannot work as databases.
   They do not understand the results they are generating,
   so they won’t know fact from fiction or recognize a natural pattern it has not been trained on.
   Therefore, LL M's are essentially next word predictors.  
2. **LLM vs Foundational Models:**  
An AI model trained on text based dataset is called LLM.
However,
   all models are also called foundational models (LLM are a subpart of it)
   which is word used to describe other AI models like image,
   video and audio etc.
3. **MultiModel AI:**  
MultiModel AI are models that can generate more than one type of data e.g., Text and image at the same time.  
4. **Parameters:**  
They are the special weights and adjustments given to **trainable part of data** on which model is trained.  
5. **Token:**  
The chunk of texts on which AI trains upon e.g. GPT-3 was trained on 300 Billion tokens and llama 1.4 Trillion tokens.
   (For reference llama had 65 Billion parameters, while GPT-3 had 175 Billion).  
6. **Quantization:**  
The process of mapping 32-bit float parameters to use smaller values like 8-bit integer.
   This process reduces the precision of weight values,
   which can lead to fall in accuracy of a model.
   However, the benefits of this are less storage is needed and computation speed increases (Faster inference).
   For a well-balanced quantized model the loss in accuracy can be very minimal while taking significantly less space and be substantially faster.  

## LLM  
### Basic Architecture:  
Taking example of a llama2 70b model, a LLM will contain 2 files, a parameter file and a run file.

A **Parameter** file (of neural network used)
will be 140GB for a 70b model(llama2 example) because it uses float16 datatype which essentially takes 2 bytes,
thus 2x70 is 140GB.

This file is an actual by-product of all the trainings done.
It can be seen as a compressed version of the multiple TB of training data along with various weights
given to each data piece.
In general, it is 100 times smaller than its trained data.

Essentially, the neural network model is being trained to learn about world data so that it can predict the next word.  

The **Run** file will be used to run the neural network model.
This process is also called **Inference**.
It can be written in any language, and its design can be understood by reading documentation of llama2.
In general, it's like 500 lines of code if written in C language.  

Since the 70b model will have 70 billion parameters,
it's really hard for anyone to know exactly how the model is working.
However, we can also tweak these parameters or their weights, during inference,
to make it better at next work prediction.  

### Training Stages  
1. The process of getting initial model created after training on dataset (~10TB of internet data compressed into neural network) is called as **pre-training**. This is very computationally intensive and needs a cluster of 1000's of GPU. The end product of this is referred to as **Base Model**.  
2. The Base model is then trained on 100K of high-quality Q&A style of responses to get an **Assistant Model**. This step can be run multiple times to iteratively improve the Assistant Model. This stage was initially primarily done by human's, but now there has been significant Human-LLM interaction 
3. Usually there is a third optional stage called as **Reinforcement learning from human feedback (RHF)**, where multiple answers are generated from the assistant model, and the best answer is chosen e.g. when writing a Poem/Haiku.  
