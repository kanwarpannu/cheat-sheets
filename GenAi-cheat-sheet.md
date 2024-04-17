# GEN AI - Generative AI Introduction

## Common terminology
1. **Models:**  
They go through large datasets in process called **Training** and find patterns and make predictions accordingly. They **DO NOT** store that information, so they cannot work as databases. They do not understand the results they are generating so they won’t know fact from fiction or recognize a natural pattern it has not been trained on. Therefore, LLM's are essentially next word predictors.  
2. **LLM vs Foundational Models:**  
An AI model trained on text based dataset is called LLM.
However, all models are also called foundational models (LLM are subpart of it) which is word used to describe other AI models like image, video and audio etc.
3. **MultiModel AI:**  
MultiModel AI are models that can generate more than one type of data eg. Text and image at same time.  

## LLM  
### Basic Architecture:  
Taking example of a llama2 70b model, a LLM will contain 2 files, a parameter file and a run file.

A parameter file (of neural network used) will be 140GB for a 70b model because it uses 2 bytes of float 16 as datatype, thus 2x70 is 140GB. This file is actual by-product of all the trainings done. It can be seen as compressed version of multiple TB's of training data. In general its 100 times smaller than its trained data. Essentially the neural network model is being trained to learn about world data so that it can predict next word.  

The run file will be used run the neural network model. This process is also called **Inference**. It can be written in any language and its design can be understood by reading documentation of llama2. In general its like 500 lines of code if written in C language.  

Since 70b model will have 70 billion parameters its really hard for anyone to know exactly how the model is working, but we can also tweak these parameters or their weights, during inference, to make it better at next work prediction.  

### Fine-Tuning  
The process of getting initial model created after training on dataset is called as **pre-training**.  
