# Optimizer comparison for NanoGPT speedrunning

Reproducible logs:
* [Adam](95a9fd44-7c13-49c7-b324-3e7d9e23a499.txt)
* [DistributedShampoo](8bfe4e35-c3fc-4b70-a984-3be937b71ff3)
* [SOAP](e21a2838-a0f2-46f2-a247-db0021165682.txt)
* [Muon](8d6193f4-27fc-4e68-899f-af70019a4d54.txt)

Results:
[1](nanogpt_speedrun81z.png)
[2](nanogpt_speedrun82z.png)

### General notes for all optimizers

All optimizers are run using zero weight decay (which is found to be empirically optimal).

In addition, in all cases we use standard Adam to optimize the embedding and lm head layers (which is also found to be empirically optimal).
Note that in the following code snippets, `raw_model.transformer.h.parameters()` gives all parameters besides those two.

## [Adam](95a9fd44-7c13-49c7-b324-3e7d9e23a499.txt)
The optimizer here is equivalent to:
```
torch.optim.Adam(raw_model.transformer.h.parameters(), lr=0.0018, betas=(0.9, 0.95))
```

We swept over:
* Learning rate
* Betas

## [DistributedShampoo](8bfe4e35-c3fc-4b70-a984-3be937b71ff3)
Run as follows:
```
DistributedShampoo(
    raw_model.transformer.h.parameters(),
    lr=0.0018,
    betas=(0.95, 0.95),
    epsilon=1e-12,
    weight_decay=0,
    max_preconditioner_dim=8192,
    precondition_frequency=10,
    use_decoupled_weight_decay=True,
    grafting_config=AdamGraftingConfig(
        beta2=0.95,
        epsilon=1e-8,
    ),   
    distributed_config=DDPShampooConfig(
        communication_dtype=CommunicationDType.FP32,
        num_trainers_per_group=8,
        communicate_params=False,
    ),   
)
```

This is using the official `DistributedShampoo` implementation from [here](https://github.com/facebookresearch/optimizers/tree/ad2809a291c01859f68fcabbcb49a2aa75fd7827/distributed_shampoo).

I was surprised at how high the wallclock overhead of this was. I presume I'm doing something wrong, but my hyperparameters appear
to be completely standard relative to the README, and after doing tens of runs I haven't found anything better.
Open to suggestions.

Things that turned out to be important:
* Don't use epsilon above 1e-8; this loses performance. Epsilon 1e-12 performs as well as 1e-15
* Betas=(0.95, 0.95) seemed optimal, which turns out to be the same thing that SOAP uses
* Higher preconditioner update frequency is better, but much slower. Infrequent updates can even lead to instability;
I was unable to find any reasonable hyperparameter settings for which update frequency 100 converges.
Using frequency 32 leads to worse performance (still better than Adam), and using frequency 3 leads to better performance
but is very slow.

I reasonably swept over the following hyperparameters:
* learning rate
* betas
* epsilon

Overall, I wasn't able to find any hyperparameters which outperform Muon in either sample-efficiency or wallclock time.

I'm open to hyperparameter suggestions; the experiment takes ~20-30 minutes to run on a fresh 8xH100 instance, so it's not hard for me to run more attempts.

## [SOAP](e21a2838-a0f2-46f2-a247-db0021165682.txt)
```
SOAP(model.transformer.h.parameters(), lr=0.0018, betas=(.95, .95), precondition_frequency=10)
```

This is using the official SOAP implementation [here](https://github.com/nikhilvyas/SOAP/blob/bbce86e890d3b697380f4376acb600c2d6c3d203/soap.py).

At update frequency 10, SOAP outperforms Shampoo, and matches Muon in terms of the final loss, but has significant wallclock and memory overhead.

Based on conversations with the authors, it is likely that a future SOAP implementation will significantly reduce the wallclock overhead, but not the memory overhead (which matters less for large-scale training).

I swept the following hyperparameters:
* learning rate
* betas

## [Muon](8d6193f4-27fc-4e68-899f-af70019a4d54.txt)
```
Muon(raw_model.transformer.h.parameters(), lr=0.02, momentum=0.95)
```

I swept the following hyperparameters:
* learning rate
* momentum


## Openness policy

These training logs are reproducible (just cut out the part besides the code, and run it using the `run.sh` in the top-level folder). They take 12-25 minutes to run.

I tried to do a good job sweeping the hyperparameters for each optimizer, but I can easily have missed something, or just not have performed enough runs.

Therefore, I am interested in any better hyperparameter settings which other researchers can find. If you post or send me your own reproducible log with
one of these optimizers, I will be very happy to boost it in any way I can.
