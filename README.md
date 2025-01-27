<br />
<div align="center">
  <h1 align="center"><img src="https://i.ibb.co/N1kYZy5/icon.png" width="30" height="30"> align-transformers</h1>
  <a href="https://arxiv.org/abs/2305.08809"><strong>Read Our Recent Paper »</strong></a>
</div>

### Tutorials
[<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/frankaging/align-transformers/blob/main/tutorials/basic_tutorials/Basic_Intervention.ipynb) [**Intervention 101**]    
[<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/frankaging/align-transformers/blob/main/tutorials/basic_tutorials/Add_New_Model_Type.ipynb) [**Add New Models and Intervention Types**]    
[<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/frankaging/align-transformers/blob/main/tutorials/advance_tutorials/Intervened_Model_Generation.ipynb) [**Intervened Model Generation**]    
[<img align="center" src="https://colab.research.google.com/assets/colab-badge.svg" />](https://colab.research.google.com/github/frankaging/align-transformers/blob/main/tutorials/advance_tutorials/DAS_with_IOI.ipynb) [**IOI Circuit with DAS**]    


# <img src="https://i.ibb.co/N1kYZy5/icon.png" width="30" height="30"> **Intervene on NN's Activations to Trace Causal Mechanism**
We have released a **new** generic library for studying model internals, which encapsulates **causal abstraction and distributed alignment search**[^ii], **path patching**[^pp], and **causal scrubbing**[^cs]. These methods were introduced recently to find or to help us find causal alignments with the internals of neural models. This library is designed as a playground for inventing new interventions, whether they're trainable or not, to uncover the causal mechanisms of neural models. Additionally, the library emphasizes scaling these methods to LLMs with billions of parameters. **This library focuses on Transformer-based models yet is extensible to other NN types as well (e.g., [MLP](https://github.com/frankaging/align-transformers/blob/main/tutorials/basic_tutorials/NonTransformer_MLP_Intervention.ipynb)).**


## Release Notes
:white_check_mark: 05/17/2023 - Preprint with the initial version of align-transformers is released!  
:white_check_mark: 10/04/2023 - Major infrastructure change to support hook-based and customizable interventions. To reproduce old experiments in [our NeurIPS 2023 paper](https://arxiv.org/abs/2305.08809), please use our shelved version [here](https://github.com/frankaging/align-transformers/releases/tag/NeurIPS-2023).   
:white_check_mark: 11/29/2023 - Released 10 tutorials so far covering different usages (on interventions, alignment, and more) of this library.

## Interventions v.s. Alignments with Model Internals
In this section, we discuss topics from interventions to alignments with model internals.

### Interventions
Intervention is the basic unit of this library. It means manipulating the model's activations, without any assumption of how the model behavior will change. We can zero-out a set of neurons, or swap activations between examples (i.e., interchange interventions). Here, we show how we can intervene in model internals with this library.

#### Loading models from HuggingFace
```py
from models.utils import create_gpt2

config, tokenizer, gpt = create_gpt2()
```

#### Create a simple alignable config
```py
alignable_config = AlignableConfig(
    alignable_model_type="gpt2",
    alignable_representations=[
        AlignableRepresentationConfig(
            0,            # intervening layer 0
            "mlp_output", # intervening mlp output
            "pos",        # intervening based on positional indices of tokens
            1             # maximally intervening one token
        ),
    ],
)
```

#### Turn the model into an alignable object
The basic idea is to consider the alignable model as a regular HuggingFace model, except that it supports an intervenable forward function.
```py
alignable_gpt = AlignableModel(alignable_config, gpt)
```

#### Intervene by swapping activations between examples
```py
base = tokenizer("The capital of Spain is", return_tensors="pt")
sources = [tokenizer("The capital of Italy is", return_tensors="pt")]

_, counterfactual_outputs = alignable_gpt(
    base,
    sources,
    {"sources->base": ([[[4]]], [[[4]]])} # intervene base with sources
)
```
--- 

### Alignments
If the model responds systematically to your interventions, then you start to associate certain regions in the network with a high-level concept. This is an alignment. Here is a more concrete example,
```py
def add_three_numbers(a, b, c):
    var_x = a + b
    return var_x + c
```
The function solves a 3-digit sum problem. Let's say, we trained a neural network to solve this problem perfectly. **One concrete alignment problem is** "Can we find the representation of (a + b) in the neural network?". We can use this library to answer this question. Specifically, we can do the following,

- **Step 1:** Form Alignment Hypothesis: We hypothesize that a set of neurons N aligns with (a + b).
- **Step 2:** Counterfactual Testings: If our hypothesis is correct, then swapping neurons N between examples would give us expected counterfactual behaviors. For instance, the values of N for (1+2)+3, when swapping with N for (2+3)+4, the output should be (2+3)+3 or (1+2)+4 depending on the direction of the swap.
- **Step 3:** Reject Sampling of Hypothesis: Running tests multiple times and aggregating statistics in terms of counterfactual behavior matching. Proposing a new hypothesis based on the results. 

To translate the above steps into API calls with the library, it will be a single call,
```py
alignable.evaluate_alignment(
    train_dataloader=test_dataloader,
    compute_metrics=compute_metrics,
    inputs_collator=inputs_collator
)
```
where you provide testing data (basically interventional data and the counterfactual behavior you are looking for) along with your metrics functions. The library will try to evaluate the alignment with the intervention you specified in the config. You can follow [this tutorial](https://github.com/frankaging/align-transformers/blob/main/tutorials/Generic%20alignment%20training.ipynb) for alignment finding and evaluation with a provided fine-tuned gpt2 model.

--- 

### Alignments with Trainable Interventions
The alignment searching process outlined above can be tedious when your neural network is large. For a single hypothesized alignment, you basically need to set up different intervention configs targeting different layers and positions to verify your hypothesis. Instead of doing this brute-force search process, you can turn it into an optimization problem which also has other benefits such as distributed alignments. For details, you can read more here[^ii].

In its crux, we basically want to train an intervention to have our desired counterfactual behaviors in mind. And if we can indeed train such interventions, we claim that causally informative information should live in the intervening representations! Below, we show one type of trainable intervention `models.interventions.RotatedSpaceIntervention` as,
```py
class RotatedSpaceIntervention(TrainableIntervention):
    
    """Intervention in the rotated space."""
    def forward(self, base, source):
        rotated_base = self.rotate_layer(base)
        rotated_source = self.rotate_layer(source)
        # interchange
        rotated_base[:self.interchange_dim] = rotated_source[:self.interchange_dim]
        # inverse base
        output = torch.matmul(rotated_base, self.rotate_layer.weight.T)
        return output
```
Instead of activation swapping in the original representation space, we first **rotate** them, and then do the swap followed by un-rotating the intervened representation. Additionally, we try to use SGD to **learn a rotation** that lets us produce expected counterfactual behavior. If we can find such rotation, we claim there is an alignment. `If the cost is between X and Y.ipynb` tutorial covers this with an advanced version of distributed alignment search, [Boundless DAS](https://arxiv.org/abs/2305.08809). There are [recent works](https://www.lesswrong.com/posts/RFtkRXHebkwxygDe2/an-interpretability-illusion-for-activation-patching-of) outlining potential limitations of doing a distributed alignment search as well.

You can now also make a single API call to train your intervention,
```py
alignable.find_alignment(
    train_dataloader=train_dataloader,
    compute_loss=compute_loss,
    compute_metrics=compute_metrics,
    inputs_collator=inputs_collator
)
```
where you need to pass in a trainable dataset, and your customized loss and metrics function. The trainable interventions can later be saved on to your disk.


## Tutorials
We released [a set of tutorials](https://github.com/frankaging/align-transformers/tree/main/tutorials) for doing model interventions and model alignments. Here are some of them,

### `Basic_Intervention.ipynb` 
(**Intervention Tutorial**) This is a tutorial for doing simple path patching as in **Path Patching**[^pp], **Causal Scrubbing**[^cs]. Thanks to [Aryaman Arora](https://aryaman.io/). This is a set of experiments trying to reproduce some of the experiments in his awesome [nano-causal-interventions](https://github.com/aryamanarora/nano-causal-interventions) repository.

### `Intervened_Model_Generation.ipynb` 
(**Intervention Tutorial**) This is a tutorial on how to intervene the TinyStories-33M model to change its story generation, with sad endings and happy endings. Different from other tutorials, this is a multi-token language generation, closer to other real-world use cases.

### `Intervention_Training.ipynb` 
(**Alignment Tutorial**) This is a tutorial covering the basics of how to train an intervention to find alignments with a gpt2 model finetuned on a logical reasoning task.

### `DAS_with_IOI.ipynb` 
(**Alignment Tutorial**) This is a tutorial reproducing key components (i.e., name mover heads, name position information) for the indirect object identification (IOI) circuit introduced by Wang et al. (2023).


## System Requirements
- Python 3.8 is supported.
- Pytorch Version: >= 2.0
- Transformers ToT is recommended
- Datasets Version ToT is recommended


## Related Works in Discovering Causal Mechanism of LLMs
If you would like to read more works on this area, here is a list of papers that try to align or discover the causal mechanisms of LLMs. 
- [Causal Abstractions of Neural Networks](https://arxiv.org/abs/2106.02997): This paper introduces interchange intervention (a.k.a. activation patching or causal scrubbing). It tries to align a causal model with the model's representations.
- [Inducing Causal Structure for Interpretable Neural Networks](https://arxiv.org/abs/2112.00826): Interchange intervention training (IIT) induces causal structures into the model's representations.
- [Localizing Model Behavior with Path Patching](https://arxiv.org/abs/2304.05969): Path patching (or causal scrubbing) to uncover causal paths in neural model.
- [Towards Automated Circuit Discovery for Mechanistic Interpretability](https://arxiv.org/abs/2304.14997): Scalable method to prune out a small set of connections in a neural network that can still complete a task.
- [Interpretability in the Wild: a Circuit for Indirect Object Identification in GPT-2 small](https://arxiv.org/abs/2211.00593): Path patching plus posthoc representation study to uncover a circuit that solves the indirect object identification (IOI) task.
- [Rigorously Assessing Natural Language Explanations of Neurons](https://arxiv.org/abs/2309.10312): Using causal abstraction to validate [neuron explanations released by OpenAI](https://openai.com/research/language-models-can-explain-neurons-in-language-models).


## Citation
If you use this repository, please consider to cite relevant papers:
```stex
  @article{geiger-etal-2023-DAS,
        title={Finding Alignments Between Interpretable Causal Variables and Distributed Neural Representations}, 
        author={Geiger, Atticus and Wu, Zhengxuan and Potts, Christopher and Icard, Thomas  and Goodman, Noah},
        year={2023},
        booktitle={arXiv}
  }

  @article{wu-etal-2023-Boundless-DAS,
        title={Interpretability at Scale: Identifying Causal Mechanisms in Alpaca}, 
        author={Wu, Zhengxuan and Geiger, Atticus and Icard, Thomas and Potts, Christopher and Goodman, Noah},
        year={2023},
        booktitle={NeurIPS}
  }
```

[^pp]: [Wang et al. (2022)](https://arxiv.org/abs/2211.00593), [Goldowsky-Dill et al. (2023)](https://arxiv.org/abs/2304.05969)
[^cs]: [Chan et al. (2022)](https://www.lesswrong.com/s/h95ayYYwMebGEYN5y)
[^ii]: [Geiger et al. (2021a)](https://arxiv.org/abs/2106.02997), [Geiger et al. (2021b)](https://arxiv.org/abs/2112.00826), [Geiger et al. (2023)](https://arxiv.org/abs/2301.04709), [Wu et al. (2023)](https://arxiv.org/pdf/2303.02536)
