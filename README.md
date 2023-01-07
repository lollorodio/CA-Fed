# Federated Learning under Heterogeneous and Correlated Client Availability

This repository is the official implementation of 
[Federated Learning under Heterogeneous and Correlated Client Availability]().

The enormous amount of data produced by mobile and IoT devices has motivated the development of federated learning (FL), a framework allowing such devices (or clients) to collaboratively train machine learning models without sharing their local data. 
FL algorithms (like FedAvg) iteratively aggregate model updates computed by clients on their own datasets. 
Clients may exhibit different levels of participation, often correlated over time and with other clients.
This paper presents the first convergence analysis for a FedAvg-like FL algorithm under heterogeneous and correlated client availability. 
Our analysis highlights how correlation adversely affects the algorithm's convergence rate and how the aggregation strategy can alleviate this effect at the cost of steering training toward a biased model.

Guided by the theoretical analysis, we propose CA-Fed, a new FL algorithm that tries to balance the conflicting goals of maximizing convergence speed and minimizing model bias, i.e., a model minimizing an objective function different from the target one.
To this purpose, CA-Fed dynamically adapts the weight given to each client and may ignore clients with low availability and large correlation.
Our experimental results show that CA-Fed achieves higher time-average accuracy and a lower standard deviation than state-of-the-art AdaFed and F3AST, both on synthetic and real datasets.

## Requirements

To install requirements:

```setup
pip install -r requirements.txt
```

## Overview 

### Federated Training 

We provide code to simulate federated training of machine learning models.
The core objects are `Client`, `ClientsSampler`, and `Aggregator`: 
different federated learning algorithms can be simulated by implementing
the local update scheme `Client.step()`, the clients' sampling strategy defined 
in `ClientsSampler.sample()`, and/or the aggregation protocol defined in 
`Aggregator.mix()` and `Aggregator.update_client()`.

We provide a Markovian clients' activity simulator, i.e., the activity 
of each client follows a Markovian model with two states, 'active' and 'inactive'. 
The logic of the simulator is implemented in `ActivitySimulator`. 
The `ClientsSampler` observes the clients' availability dynamics generated by the
`ActivitySimulator` with the procedure `ClientsSampler.get_active_clients()`.
The configuration parameters storing the clients' activity dynamics are in 
`data\constants.py` and `data\main.py`. For more advanced customization,
please refer to `data\README.md`.

Once the `ActivitySimulator` has been generated, the aggregator can keep track
of the clients' activity dynamic through the `ActivityEstimator`, that is
responsible to estimate the availability and stability parameters for each client.
One can finally design algorithms that are aware of the heterogeneous 
and correlated clients' availability, defining suitable client sampling and 
aggregation strategies in `ClientsSampler.sample()` and `Aggregator.mix()`.

A baseline example for simulating a federated training using FedAvg is
provided in [examples/FedAvg.md](examples/FedAvg.md).

### Algorithms

In addition to the baseline that assigns aggregation weights to the clients
inversely proportional to their estimated availability (\pi_k)_{k \in \mathcal{K}, 
that we refer in the paper as `Unbiased` aggregation strategy, this repository supports
the following federated learning algorithms:
* FedAvg ([McMahan et al. 2017](http://proceedings.mlr.press/v54/mcmahan17a?ref=https://githubhelp.com))
* AdaFed ([Tan et al. 2022](https://ieeexplore.ieee.org/abstract/document/9762058))
* F3AST ([Ribero et al. 2022](https://arxiv.org/abs/2205.06730))
* CA-Fed ([Correlation-Aware Federated Learning, Ours]())

To execute the different algorithms, refer to the classes `UnbiasedClientsSampler`, 
`AdaFedClientsSampler`, `F3AST`, and `MarkovianClientsSampler`. Alternatively, you can
execute the python file `run_experiment.py` passing 
{`unbiased`, `adafed`, `fast`, or `markov`}
as values of the `client_sampler` argument,
or calling the method `utils.get_clients_sampler` with the name of the algorithm
as value of the `sampler_type` argument.

Please refer to the section `paper_experiments/` for practical examples.

### Datasets

We provide four federated benchmarks spanning different machine learning tasks: 
image classification (CIFAR10), handwritten character recognition (MNIST), in
addition to two synthetic dataset (Clustered and LEAF), for binary and multi-class
classification.

The following table summarizes the datasets and models:

| Dataset                                                | Task                              |  Model |
|--------------------------------------------------------|-----------------------------------|------- |
| Synthetic Clustered                                    | Binary classification             | Linear model | 
| [Synthetic LEAF](https://leaf.cmu.edu/)                | Multi-class classification        | Linear model  |
| [MNIST](http://yann.lecun.com/exdb/mnist/)             | Handwritten character recognition | Linear model    |
| [CIFAR10](https://www.cs.toronto.edu/~kriz/cifar.html) | Image classification              | 2-layer CNN + 2-layer FFN |

See `data/README.md` for instructions on generating data and 
simulation configuration file.

## Paper Experiments
Scripts to reproduce the paper experiments are provided in `paper_experiments/`.
Specify the name of the dataset (experiment), the used algorithm 
(client sampler), and configure all other hyper-parameters (please refer to their
values as specified in the paper).
An example on one dataset (MNIST), with a specific choice of federated learning method
(CA-Fed), is as follows:

```
echo "==> Run experiment with 'markov' clients sampler"

name="markov"

python run_experiment.py \
  --experiment mnist \
  --cfg_file_path data/mnist/cfg.json \
  --objective_type weighted \
  --aggregator_type centralized \
  --clients_sampler "${name}" \
  --smoothness_param 0.0 \
  --tolerance_param 0.0 \
  --n_rounds 150 \
  --local_steps 5 \
  --local_optimizer sgd \
  --local_lr 0.03 \
  --server_optimizer sgd \
  --server_lr 0.03 \
  --train_bz 64 \
  --test_bz 1024 \
  --device cuda \
  --log_freq 5 \
  --verbose 1 \
  --seed 1234 \
  --logs_dir "logs/mnist/activity_markov/mnist/seed_1234" \
  --history_path "history/mnist/activity_markov/mnist/seed_1234.json" 
```

Please refer to more examples in `paper_experiments/`.

## Plots

To generate the plots run: 

```
python make_plots.py \
    --logs_dir <logs_dir> \
    --history_dir <history_dir> \
    --save_dir <save_dir>
```

## Results

The performance of each aggregation strategy (which consider all clients in the 
case of FedAvg and AdaFed, samples clients in the case of F3AST and CA-Fed) is 
evaluated on the local test dataset (unseen at training). The following table 
shows for each algorithm: the average over three runs of the maximum test accuracy 
achieved during training, the time-average test accuracy achieved during training,
together with its standard deviation within the second half of the training period,
on the Synthetic Clustered and MNIST datasets:

| Aggregation Strategy | Maximum       | Time-Average  | Deviation   |
|----------------------|---------------|---------------|-------------|
| **Unbiased**         | 78.94 / 64.87 | 75.33 / 61.39 | 0.48 / 1.09 |
| **F3AST**            | 78.97 / 64.91 | 75.33 / 61.52 | 0.40 / 0.94 |
| **AdaFed**           | 78.69 / 63.77 | 74.81 / 60.48 | 0.59 / 1.37 |
| **CA-Fed (Ours)**    | 79.03 / 64.94 | 76.22 / 62.76 | 0.28 / 0.61 |

We can also see the time-average accuracy up to round $t$ of the learned
model averaged over three different runs:

![](https://user-images.githubusercontent.com/45441112/211089920-dba2eb1e-98d7-4da6-8ab4-542b67790faa.png)

Similar plots can be built for other experiments using the `make_plot` function
in `utils.plots.py`.

## Citation

If you use our code or wish to refer to our results,
please use the following BibTex entry:

```commandline

```
