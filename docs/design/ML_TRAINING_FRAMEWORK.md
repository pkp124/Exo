# ML Training Framework Design

**Training AI Models on Execution Traces**

Version: 0.1.0-draft  
Status: Design Proposal  
Last Updated: January 2026

---

## Table of Contents

1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Data Pipeline](#data-pipeline)
4. [Model Architectures](#model-architectures)
5. [Training Infrastructure](#training-infrastructure)
6. [Model Registry](#model-registry)
7. [Inference API](#inference-api)
8. [Continuous Learning](#continuous-learning)

---

## Overview

The ML Training Framework enables learning predictive models from execution traces. These models power the architecture exploration platform, enabling:

- **Performance prediction** without running on actual hardware
- **Bottleneck identification** before implementation
- **Architecture recommendations** based on workload characteristics
- **Anomaly detection** for regression monitoring

### Training Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Training Data Flow                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Trace Collection         Feature Engineering         Model Training        │
│  ────────────────         ───────────────────         ──────────────        │
│                                                                              │
│  ┌─────────────┐         ┌─────────────────┐         ┌────────────────┐    │
│  │ SoC A       │         │                 │         │                │    │
│  │ - Model X   │────┐    │  Feature        │         │  Performance   │    │
│  │ - Traces    │    │    │  Extraction     │         │  Predictor     │    │
│  └─────────────┘    │    │                 │         │                │    │
│                     │    │  ┌───────────┐  │         │  ┌──────────┐  │    │
│  ┌─────────────┐    │    │  │ Model     │  │         │  │ XGBoost  │  │    │
│  │ SoC B       │    ├───▶│  │ Features  │──┼────────▶│  │ or       │  │    │
│  │ - Model Y   │────┤    │  ├───────────┤  │         │  │ GNN      │  │    │
│  │ - Traces    │    │    │  │ Hardware  │  │         │  │ or       │  │    │
│  └─────────────┘    │    │  │ Features  │  │         │  │ Ensemble │  │    │
│                     │    │  ├───────────┤  │         │  └──────────┘  │    │
│  ┌─────────────┐    │    │  │ Graph     │  │         │                │    │
│  │ SoC C       │    │    │  │ Embedding │  │         │  Targets:      │    │
│  │ - Model Z   │────┘    │  └───────────┘  │         │  - Latency     │    │
│  │ - Traces    │         │                 │         │  - Power       │    │
│  └─────────────┘         └─────────────────┘         │  - Utilization │    │
│                                                       └────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Learning Objectives

### Primary Tasks

| Task | Input | Output | Use Case |
|------|-------|--------|----------|
| **Latency Prediction** | Model + Hardware spec | Predicted latency (ms) | Architecture exploration |
| **Throughput Prediction** | Model + Hardware + Batch | Throughput (inferences/sec) | Capacity planning |
| **Power Prediction** | Model + Hardware + Latency | Power consumption (W) | Power budgeting |
| **Bottleneck Classification** | Model + Hardware + Trace | Bottleneck type | Optimization guidance |
| **Operator Time Prediction** | Operator + Hardware | Per-op latency (μs) | Fine-grained analysis |
| **Partitioning Scoring** | Partition + Hardware | Partition quality score | Partitioning optimization |

### Secondary Tasks

| Task | Input | Output | Use Case |
|------|-------|--------|----------|
| **Anomaly Detection** | New trace + Historical | Anomaly score | Regression detection |
| **Similar Trace Retrieval** | Query trace | Similar traces | Knowledge transfer |
| **Hardware Recommendation** | Model + Constraints | Ranked hardware list | Hardware selection |
| **Optimization Suggestion** | Trace + Bottleneck | Suggested optimizations | Guided optimization |

---

## Data Pipeline

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ML Data Pipeline                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Stage 1: Ingestion                            │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │ Perfetto   │  │ OTLP       │  │ Exo Binary │  │ CSV/JSON   │    │   │
│  │  │ Traces     │  │ Traces     │  │ Traces     │  │ Metadata   │    │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘    │   │
│  │        └───────────────┴───────────────┴───────────────┘            │   │
│  │                              │                                       │   │
│  └──────────────────────────────┼───────────────────────────────────────┘   │
│                                 ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Stage 2: Parsing                              │   │
│  │  - Parse trace format                                                │   │
│  │  - Extract spans, events, counters                                  │   │
│  │  - Build operator graph                                              │   │
│  │  - Validate schema                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                 │                                            │
│                                 ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Stage 3: Feature Extraction                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │   │
│  │  │ Model Features  │  │ HW Features     │  │ Graph Features  │     │   │
│  │  │ - op_counts     │  │ - compute_tops  │  │ - node_embed    │     │   │
│  │  │ - total_flops   │  │ - memory_bw     │  │ - graph_embed   │     │   │
│  │  │ - memory_size   │  │ - cache_size    │  │ - topology      │     │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                 │                                            │
│                                 ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Stage 4: Label Extraction                     │   │
│  │  - Total latency (from trace)                                       │   │
│  │  - Per-operator latency                                              │   │
│  │  - Utilization metrics                                               │   │
│  │  - Power measurements (if available)                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                 │                                            │
│                                 ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Stage 5: Dataset Creation                     │   │
│  │  - Train/validation/test split                                      │   │
│  │  - Stratification by SoC, model type                                │   │
│  │  - Normalization/scaling                                             │   │
│  │  - Save to efficient format (Parquet, TFRecord)                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Feature Definitions

```python
from dataclasses import dataclass
from typing import List, Dict
import numpy as np

@dataclass
class ModelFeatures:
    """Features extracted from the AI model."""
    
    # Operator statistics
    num_operators: int
    op_type_counts: Dict[str, int]  # e.g., {"conv2d": 10, "matmul": 5}
    op_type_ratios: Dict[str, float]
    
    # Compute characteristics
    total_flops: int
    total_macs: int
    total_params: int
    
    # Memory characteristics
    total_weight_bytes: int
    total_activation_bytes: int
    peak_activation_bytes: int
    
    # Tensor statistics
    avg_tensor_size: float
    max_tensor_size: int
    tensor_dtype: str
    
    # Arithmetic intensity
    avg_arithmetic_intensity: float
    min_arithmetic_intensity: float
    max_arithmetic_intensity: float
    
    # Graph structure
    graph_depth: int
    graph_width: int
    num_branches: int
    parallelism_degree: float
    
    def to_vector(self) -> np.ndarray:
        """Convert to fixed-size feature vector."""
        return np.array([
            self.num_operators,
            self.total_flops,
            self.total_macs,
            self.total_params,
            self.total_weight_bytes,
            self.total_activation_bytes,
            self.peak_activation_bytes,
            self.avg_tensor_size,
            self.max_tensor_size,
            self.avg_arithmetic_intensity,
            self.min_arithmetic_intensity,
            self.max_arithmetic_intensity,
            self.graph_depth,
            self.graph_width,
            self.num_branches,
            self.parallelism_degree,
            # Op type ratios (fixed order)
            self.op_type_ratios.get('conv2d', 0),
            self.op_type_ratios.get('matmul', 0),
            self.op_type_ratios.get('attention', 0),
            self.op_type_ratios.get('relu', 0),
            self.op_type_ratios.get('softmax', 0),
            self.op_type_ratios.get('layernorm', 0),
            self.op_type_ratios.get('pooling', 0),
            self.op_type_ratios.get('other', 0),
        ])


@dataclass
class HardwareFeatures:
    """Features describing hardware configuration."""
    
    # Compute
    npu_tops: float  # INT8 TOPS
    npu_tflops: float  # FP16 TFLOPS
    npu_units: int
    npu_frequency_ghz: float
    
    # Memory
    memory_bandwidth_gbps: float
    sram_size_kb: int
    cache_size_kb: int
    dram_size_gb: float
    
    # DMA
    dma_channels: int
    dma_bandwidth_gbps: float
    
    # Other
    cpu_cores: int
    cpu_frequency_ghz: float
    has_gpu: bool
    
    def to_vector(self) -> np.ndarray:
        """Convert to fixed-size feature vector."""
        return np.array([
            self.npu_tops,
            self.npu_tflops,
            self.npu_units,
            self.npu_frequency_ghz,
            self.memory_bandwidth_gbps,
            self.sram_size_kb,
            self.cache_size_kb,
            self.dram_size_gb,
            self.dma_channels,
            self.dma_bandwidth_gbps,
            self.cpu_cores,
            self.cpu_frequency_ghz,
            float(self.has_gpu),
        ])


@dataclass  
class OperatorFeatures:
    """Features for a single operator (for GNN)."""
    
    op_type_embedding: np.ndarray  # One-hot or learned
    flops: int
    macs: int
    memory_read_bytes: int
    memory_write_bytes: int
    arithmetic_intensity: float
    input_shape: List[int]
    output_shape: List[int]
    is_on_critical_path: bool
    num_predecessors: int
    num_successors: int
    
    def to_vector(self) -> np.ndarray:
        """Convert to node feature vector."""
        return np.concatenate([
            self.op_type_embedding,
            np.array([
                np.log1p(self.flops),
                np.log1p(self.macs),
                np.log1p(self.memory_read_bytes),
                np.log1p(self.memory_write_bytes),
                self.arithmetic_intensity,
                float(self.is_on_critical_path),
                self.num_predecessors,
                self.num_successors,
                np.prod(self.input_shape),
                np.prod(self.output_shape),
            ])
        ])


@dataclass
class TrainingExample:
    """Complete training example."""
    
    # Identifiers
    trace_id: str
    soc_id: str
    model_id: str
    
    # Features
    model_features: ModelFeatures
    hardware_features: HardwareFeatures
    operator_features: List[OperatorFeatures]  # For GNN
    graph_edges: List[tuple]  # (src, dst) edges
    
    # Labels
    total_latency_ms: float
    per_operator_latency_us: Dict[str, float]
    npu_utilization: float
    memory_bandwidth_utilization: float
    power_mw: float
    bottleneck_type: str  # "compute", "memory", "dma", etc.
    
    # Metadata
    batch_size: int
    firmware_version: str
    collection_timestamp: str
```

### Data Augmentation

```python
class TraceAugmenter:
    """Augment traces to increase training data diversity."""
    
    def augment(self, example: TrainingExample) -> List[TrainingExample]:
        augmented = [example]
        
        # Batch size scaling (if we have multi-batch traces)
        if example.batch_size == 1:
            # Estimate larger batch performance
            for batch_size in [2, 4, 8]:
                aug = self._scale_batch(example, batch_size)
                augmented.append(aug)
        
        # Hardware interpolation
        for scale in [0.8, 1.2]:
            aug = self._scale_hardware(example, scale)
            augmented.append(aug)
        
        return augmented
    
    def _scale_batch(self, example, new_batch_size):
        """Estimate performance for different batch size."""
        scale = new_batch_size / example.batch_size
        
        new_example = copy.deepcopy(example)
        new_example.batch_size = new_batch_size
        
        # Simple scaling model (can be improved)
        # Latency scales sub-linearly due to batching efficiency
        new_example.total_latency_ms *= scale ** 0.7
        
        # Memory scales linearly
        new_example.model_features.peak_activation_bytes *= scale
        
        return new_example
    
    def _scale_hardware(self, example, scale):
        """Create synthetic example with scaled hardware."""
        new_example = copy.deepcopy(example)
        
        # Scale hardware
        new_example.hardware_features.npu_tops *= scale
        new_example.hardware_features.memory_bandwidth_gbps *= scale
        
        # Estimate new performance (simple model)
        if example.bottleneck_type == "compute":
            # Compute-bound: scales with compute
            new_example.total_latency_ms /= scale
        elif example.bottleneck_type == "memory":
            # Memory-bound: scales with bandwidth
            new_example.total_latency_ms /= scale
        else:
            # Mixed: partial scaling
            new_example.total_latency_ms /= (scale ** 0.5)
        
        return new_example
```

---

## Model Architectures

### Architecture 1: Gradient Boosted Trees (Baseline)

Fast, interpretable, works well for tabular features.

```python
from xgboost import XGBRegressor
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

class XGBoostPredictor:
    """XGBoost-based performance predictor."""
    
    def __init__(self, config=None):
        self.config = config or {
            'n_estimators': 500,
            'max_depth': 8,
            'learning_rate': 0.05,
            'subsample': 0.8,
            'colsample_bytree': 0.8,
            'min_child_weight': 5,
            'reg_alpha': 0.1,
            'reg_lambda': 1.0,
        }
        
        self.model = Pipeline([
            ('scaler', StandardScaler()),
            ('xgb', XGBRegressor(**self.config))
        ])
    
    def prepare_features(self, examples: List[TrainingExample]) -> np.ndarray:
        """Combine model and hardware features."""
        features = []
        for ex in examples:
            feature_vec = np.concatenate([
                ex.model_features.to_vector(),
                ex.hardware_features.to_vector(),
                np.array([ex.batch_size])
            ])
            features.append(feature_vec)
        return np.array(features)
    
    def train(self, examples: List[TrainingExample]):
        X = self.prepare_features(examples)
        y = np.array([ex.total_latency_ms for ex in examples])
        
        self.model.fit(X, y)
    
    def predict(self, examples: List[TrainingExample]) -> np.ndarray:
        X = self.prepare_features(examples)
        return self.model.predict(X)
    
    def get_feature_importance(self) -> Dict[str, float]:
        """Get feature importance for interpretability."""
        xgb_model = self.model.named_steps['xgb']
        importance = xgb_model.feature_importances_
        
        feature_names = self._get_feature_names()
        return dict(zip(feature_names, importance))
```

### Architecture 2: Graph Neural Network

Better for capturing operator graph structure.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch_geometric.nn import GCNConv, GATConv, global_mean_pool, global_max_pool
from torch_geometric.data import Data, Batch

class OperatorGNN(nn.Module):
    """GNN for encoding operator graphs."""
    
    def __init__(self, node_features, hidden_dim=128, num_layers=4):
        super().__init__()
        
        self.node_encoder = nn.Linear(node_features, hidden_dim)
        
        # Graph convolution layers
        self.convs = nn.ModuleList()
        for _ in range(num_layers):
            self.convs.append(GATConv(hidden_dim, hidden_dim, heads=4, concat=False))
        
        self.norms = nn.ModuleList()
        for _ in range(num_layers):
            self.norms.append(nn.LayerNorm(hidden_dim))
        
        # Readout: combine mean and max pooling
        self.graph_encoder = nn.Linear(hidden_dim * 2, hidden_dim)
    
    def forward(self, data):
        x, edge_index, batch = data.x, data.edge_index, data.batch
        
        # Encode nodes
        x = self.node_encoder(x)
        x = F.relu(x)
        
        # Message passing
        for conv, norm in zip(self.convs, self.norms):
            x_new = conv(x, edge_index)
            x_new = norm(x_new)
            x_new = F.relu(x_new)
            x = x + x_new  # Residual connection
        
        # Graph-level readout
        mean_pool = global_mean_pool(x, batch)
        max_pool = global_max_pool(x, batch)
        graph_repr = torch.cat([mean_pool, max_pool], dim=1)
        graph_repr = self.graph_encoder(graph_repr)
        
        return graph_repr


class GNNPerformancePredictor(nn.Module):
    """Complete performance predictor using GNN."""
    
    def __init__(self, node_features, hw_features, hidden_dim=128):
        super().__init__()
        
        # Graph encoder for operator graph
        self.gnn = OperatorGNN(node_features, hidden_dim)
        
        # Hardware feature encoder
        self.hw_encoder = nn.Sequential(
            nn.Linear(hw_features, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        
        # Combined prediction head
        self.predictor = nn.Sequential(
            nn.Linear(hidden_dim * 2, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(hidden_dim, hidden_dim // 2),
            nn.ReLU(),
            nn.Linear(hidden_dim // 2, 1)
        )
    
    def forward(self, graph_data, hw_features):
        # Encode graph
        graph_repr = self.gnn(graph_data)
        
        # Encode hardware
        hw_repr = self.hw_encoder(hw_features)
        
        # Combine and predict
        combined = torch.cat([graph_repr, hw_repr], dim=1)
        prediction = self.predictor(combined)
        
        return prediction.squeeze(-1)


class GNNTrainer:
    """Trainer for GNN-based predictor."""
    
    def __init__(self, model, lr=1e-3, weight_decay=1e-4):
        self.model = model
        self.optimizer = torch.optim.AdamW(model.parameters(), lr=lr, weight_decay=weight_decay)
        self.scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(self.optimizer, T_max=100)
        self.loss_fn = nn.MSELoss()
    
    def train_epoch(self, dataloader):
        self.model.train()
        total_loss = 0
        
        for batch in dataloader:
            self.optimizer.zero_grad()
            
            predictions = self.model(batch.graph_data, batch.hw_features)
            loss = self.loss_fn(predictions, batch.latency)
            
            loss.backward()
            torch.nn.utils.clip_grad_norm_(self.model.parameters(), 1.0)
            self.optimizer.step()
            
            total_loss += loss.item()
        
        self.scheduler.step()
        return total_loss / len(dataloader)
    
    def evaluate(self, dataloader):
        self.model.eval()
        predictions = []
        targets = []
        
        with torch.no_grad():
            for batch in dataloader:
                pred = self.model(batch.graph_data, batch.hw_features)
                predictions.extend(pred.cpu().numpy())
                targets.extend(batch.latency.cpu().numpy())
        
        predictions = np.array(predictions)
        targets = np.array(targets)
        
        mape = np.mean(np.abs(predictions - targets) / targets) * 100
        rmse = np.sqrt(np.mean((predictions - targets) ** 2))
        r2 = 1 - np.sum((predictions - targets) ** 2) / np.sum((targets - np.mean(targets)) ** 2)
        
        return {'mape': mape, 'rmse': rmse, 'r2': r2}
```

### Architecture 3: Ensemble Model

Combine multiple models for best performance.

```python
class EnsemblePredictor:
    """Ensemble of different model types."""
    
    def __init__(self):
        self.models = {
            'xgboost': XGBoostPredictor(),
            'gnn': GNNPerformancePredictor(node_features=32, hw_features=13),
            'linear': LinearRegression(),
        }
        
        # Meta-learner to combine predictions
        self.meta_learner = XGBRegressor(n_estimators=100, max_depth=3)
        self.weights = None
    
    def train(self, train_examples, val_examples):
        # Train base models
        for name, model in self.models.items():
            print(f"Training {name}...")
            model.train(train_examples)
        
        # Get predictions on validation set for meta-learning
        val_preds = {}
        for name, model in self.models.items():
            val_preds[name] = model.predict(val_examples)
        
        # Stack predictions
        X_meta = np.column_stack([val_preds[name] for name in self.models.keys()])
        y_meta = np.array([ex.total_latency_ms for ex in val_examples])
        
        # Train meta-learner
        self.meta_learner.fit(X_meta, y_meta)
    
    def predict(self, examples):
        # Get base predictions
        base_preds = {}
        for name, model in self.models.items():
            base_preds[name] = model.predict(examples)
        
        # Stack and meta-predict
        X_meta = np.column_stack([base_preds[name] for name in self.models.keys()])
        return self.meta_learner.predict(X_meta)
```

### Architecture 4: Per-Operator Predictor

Predict each operator individually, then sum.

```python
class OperatorLevelPredictor:
    """Predict performance at operator level."""
    
    def __init__(self):
        # Separate model for each operator type
        self.op_models = {}
        self.overhead_model = None
    
    def train(self, examples: List[TrainingExample]):
        # Group operator data by type
        op_data = defaultdict(list)
        for ex in examples:
            for op_name, latency_us in ex.per_operator_latency_us.items():
                op_type = self._get_op_type(op_name)
                op_features = self._get_op_features(ex, op_name)
                hw_features = ex.hardware_features.to_vector()
                
                op_data[op_type].append({
                    'features': np.concatenate([op_features, hw_features]),
                    'latency_us': latency_us
                })
        
        # Train per-operator models
        for op_type, data in op_data.items():
            X = np.array([d['features'] for d in data])
            y = np.array([d['latency_us'] for d in data])
            
            model = XGBRegressor(n_estimators=200, max_depth=6)
            model.fit(X, y)
            self.op_models[op_type] = model
        
        # Train overhead model (for non-operator time)
        overhead_data = []
        for ex in examples:
            predicted_op_time = sum(ex.per_operator_latency_us.values()) / 1000
            actual_total = ex.total_latency_ms
            overhead = actual_total - predicted_op_time
            
            overhead_data.append({
                'features': np.concatenate([
                    ex.model_features.to_vector(),
                    ex.hardware_features.to_vector()
                ]),
                'overhead_ms': overhead
            })
        
        X_overhead = np.array([d['features'] for d in overhead_data])
        y_overhead = np.array([d['overhead_ms'] for d in overhead_data])
        
        self.overhead_model = XGBRegressor(n_estimators=100, max_depth=4)
        self.overhead_model.fit(X_overhead, y_overhead)
    
    def predict(self, example: TrainingExample) -> Dict[str, float]:
        """Predict per-operator and total latency."""
        per_op_predictions = {}
        
        # Predict each operator
        for op_features in example.operator_features:
            op_type = op_features.op_type
            if op_type in self.op_models:
                features = np.concatenate([
                    op_features.to_vector(),
                    example.hardware_features.to_vector()
                ]).reshape(1, -1)
                
                pred_us = self.op_models[op_type].predict(features)[0]
                per_op_predictions[op_features.op_name] = pred_us
        
        # Predict overhead
        overhead_features = np.concatenate([
            example.model_features.to_vector(),
            example.hardware_features.to_vector()
        ]).reshape(1, -1)
        overhead_ms = self.overhead_model.predict(overhead_features)[0]
        
        # Total prediction
        total_op_time_ms = sum(per_op_predictions.values()) / 1000
        total_ms = total_op_time_ms + overhead_ms
        
        return {
            'total_latency_ms': total_ms,
            'per_operator_latency_us': per_op_predictions,
            'overhead_ms': overhead_ms
        }
```

---

## Training Infrastructure

### Training Pipeline

```python
from dataclasses import dataclass
from typing import Optional
import mlflow

@dataclass
class TrainingConfig:
    """Configuration for model training."""
    
    # Data
    data_source: str  # Path or database connection
    train_split: float = 0.7
    val_split: float = 0.15
    test_split: float = 0.15
    stratify_by: List[str] = None  # e.g., ["soc_id", "model_architecture"]
    
    # Features
    feature_config: Dict[str, Any] = None
    
    # Model
    model_type: str = "xgboost"  # "xgboost", "gnn", "ensemble"
    model_config: Dict[str, Any] = None
    
    # Training
    epochs: int = 100
    batch_size: int = 32
    learning_rate: float = 1e-3
    early_stopping_patience: int = 10
    
    # Experiment tracking
    experiment_name: str = "performance_prediction"
    run_name: Optional[str] = None


class TrainingPipeline:
    """End-to-end training pipeline."""
    
    def __init__(self, config: TrainingConfig):
        self.config = config
        self.data_loader = None
        self.model = None
        self.metrics = {}
    
    def run(self):
        """Execute full training pipeline."""
        
        with mlflow.start_run(run_name=self.config.run_name):
            # Log config
            mlflow.log_params(self._flatten_config())
            
            # Load and prepare data
            print("Loading data...")
            train, val, test = self._load_data()
            
            mlflow.log_metric("train_size", len(train))
            mlflow.log_metric("val_size", len(val))
            mlflow.log_metric("test_size", len(test))
            
            # Create model
            print("Creating model...")
            self.model = self._create_model()
            
            # Train
            print("Training...")
            self._train(train, val)
            
            # Evaluate
            print("Evaluating...")
            self.metrics = self._evaluate(test)
            
            for name, value in self.metrics.items():
                mlflow.log_metric(name, value)
            
            # Save model
            print("Saving model...")
            self._save_model()
            
            print(f"Training complete. Test MAPE: {self.metrics['mape']:.2f}%")
            
        return self.model, self.metrics
    
    def _load_data(self):
        """Load and split data."""
        from exo_platform.data import TraceDataset
        
        dataset = TraceDataset.load(
            self.config.data_source,
            feature_config=self.config.feature_config
        )
        
        train, val, test = dataset.split(
            train=self.config.train_split,
            val=self.config.val_split,
            test=self.config.test_split,
            stratify_by=self.config.stratify_by
        )
        
        return train, val, test
    
    def _create_model(self):
        """Create model based on config."""
        if self.config.model_type == "xgboost":
            return XGBoostPredictor(self.config.model_config)
        elif self.config.model_type == "gnn":
            return GNNPerformancePredictor(**self.config.model_config)
        elif self.config.model_type == "ensemble":
            return EnsemblePredictor()
        else:
            raise ValueError(f"Unknown model type: {self.config.model_type}")
    
    def _train(self, train, val):
        """Train the model."""
        if hasattr(self.model, 'train_with_validation'):
            self.model.train_with_validation(
                train, val,
                epochs=self.config.epochs,
                patience=self.config.early_stopping_patience
            )
        else:
            self.model.train(train)
    
    def _evaluate(self, test):
        """Evaluate on test set."""
        predictions = self.model.predict(test)
        targets = np.array([ex.total_latency_ms for ex in test])
        
        mape = np.mean(np.abs(predictions - targets) / targets) * 100
        rmse = np.sqrt(np.mean((predictions - targets) ** 2))
        mae = np.mean(np.abs(predictions - targets))
        r2 = 1 - np.sum((predictions - targets) ** 2) / np.sum((targets - np.mean(targets)) ** 2)
        
        return {
            'mape': mape,
            'rmse': rmse,
            'mae': mae,
            'r2': r2
        }
    
    def _save_model(self):
        """Save model artifacts."""
        model_path = f"models/{self.config.experiment_name}/{mlflow.active_run().info.run_id}"
        self.model.save(model_path)
        mlflow.log_artifact(model_path)
```

### Distributed Training

```python
class DistributedTrainer:
    """Distributed training for large datasets."""
    
    def __init__(self, config, num_workers=4):
        self.config = config
        self.num_workers = num_workers
    
    def train(self):
        import ray
        from ray import train
        from ray.train import ScalingConfig
        from ray.train.xgboost import XGBoostTrainer
        
        ray.init()
        
        # For XGBoost
        trainer = XGBoostTrainer(
            scaling_config=ScalingConfig(
                num_workers=self.num_workers,
                use_gpu=False,
            ),
            label_column="latency_ms",
            params=self.config.model_config,
            datasets={"train": train_dataset, "valid": val_dataset},
        )
        
        result = trainer.fit()
        return result.checkpoint
```

---

## Model Registry

### Registry Design

```python
from datetime import datetime
from enum import Enum

class ModelStage(Enum):
    DEVELOPMENT = "development"
    STAGING = "staging"
    PRODUCTION = "production"
    ARCHIVED = "archived"

@dataclass
class ModelVersion:
    """A version of a trained model."""
    
    model_id: str
    version: int
    stage: ModelStage
    
    # Training info
    trained_at: datetime
    training_data_version: str
    training_config: Dict
    
    # Metrics
    metrics: Dict[str, float]
    
    # Artifacts
    model_path: str
    feature_config_path: str
    
    # Metadata
    description: str
    created_by: str

class ModelRegistry:
    """Registry for trained models."""
    
    def __init__(self, storage_path: str):
        self.storage_path = storage_path
        self.db = self._init_db()
    
    def register_model(
        self,
        model_id: str,
        model: Any,
        training_config: Dict,
        metrics: Dict[str, float],
        description: str = ""
    ) -> ModelVersion:
        """Register a new model version."""
        
        # Get next version number
        version = self._get_next_version(model_id)
        
        # Save model artifacts
        model_path = f"{self.storage_path}/{model_id}/v{version}/model.pkl"
        self._save_model(model, model_path)
        
        # Create version record
        model_version = ModelVersion(
            model_id=model_id,
            version=version,
            stage=ModelStage.DEVELOPMENT,
            trained_at=datetime.now(),
            training_data_version=training_config.get('data_version'),
            training_config=training_config,
            metrics=metrics,
            model_path=model_path,
            feature_config_path=f"{self.storage_path}/{model_id}/v{version}/features.json",
            description=description,
            created_by=os.getenv('USER', 'unknown')
        )
        
        self._save_version_record(model_version)
        
        return model_version
    
    def get_production_model(self, model_id: str) -> Tuple[Any, ModelVersion]:
        """Get the production version of a model."""
        version = self._get_latest_by_stage(model_id, ModelStage.PRODUCTION)
        model = self._load_model(version.model_path)
        return model, version
    
    def promote_to_production(self, model_id: str, version: int):
        """Promote a model version to production."""
        # Demote current production
        current_prod = self._get_latest_by_stage(model_id, ModelStage.PRODUCTION)
        if current_prod:
            self._update_stage(model_id, current_prod.version, ModelStage.ARCHIVED)
        
        # Promote new version
        self._update_stage(model_id, version, ModelStage.PRODUCTION)
    
    def compare_versions(
        self,
        model_id: str,
        version_a: int,
        version_b: int
    ) -> Dict:
        """Compare two model versions."""
        v_a = self._get_version(model_id, version_a)
        v_b = self._get_version(model_id, version_b)
        
        comparison = {
            'metrics_comparison': {
                metric: {
                    'version_a': v_a.metrics.get(metric),
                    'version_b': v_b.metrics.get(metric),
                    'diff': v_b.metrics.get(metric, 0) - v_a.metrics.get(metric, 0)
                }
                for metric in set(v_a.metrics.keys()) | set(v_b.metrics.keys())
            },
            'config_diff': self._diff_configs(v_a.training_config, v_b.training_config)
        }
        
        return comparison
```

---

## Inference API

### Python API

```python
from exo_platform.predict import PerformancePredictor

# Load production model
predictor = PerformancePredictor.load_production("latency_predictor")

# Predict for new scenario
prediction = predictor.predict(
    model=ModelSpec.from_onnx("resnet50.onnx"),
    hardware=HardwareSpec(
        npu_tops=20,
        memory_bandwidth_gbps=100,
        cache_size_mb=4
    ),
    batch_size=1
)

print(f"Predicted latency: {prediction.latency_ms:.1f}ms")
print(f"Confidence interval: [{prediction.ci_lower:.1f}, {prediction.ci_upper:.1f}]ms")
print(f"Predicted bottleneck: {prediction.bottleneck}")
```

### REST API

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PredictionRequest(BaseModel):
    model_onnx_path: str = None
    model_spec: dict = None
    hardware_spec: dict
    batch_size: int = 1

class PredictionResponse(BaseModel):
    latency_ms: float
    confidence_interval: tuple
    bottleneck: str
    per_operator_latency: dict = None

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    predictor = PerformancePredictor.load_production("latency_predictor")
    
    if request.model_onnx_path:
        model = ModelSpec.from_onnx(request.model_onnx_path)
    else:
        model = ModelSpec.from_dict(request.model_spec)
    
    hardware = HardwareSpec.from_dict(request.hardware_spec)
    
    prediction = predictor.predict(
        model=model,
        hardware=hardware,
        batch_size=request.batch_size
    )
    
    return PredictionResponse(
        latency_ms=prediction.latency_ms,
        confidence_interval=(prediction.ci_lower, prediction.ci_upper),
        bottleneck=prediction.bottleneck,
        per_operator_latency=prediction.per_operator_latency
    )
```

---

## Continuous Learning

### Online Learning Pipeline

```python
class ContinuousLearner:
    """Continuously improve models with new traces."""
    
    def __init__(self, model_id: str, registry: ModelRegistry):
        self.model_id = model_id
        self.registry = registry
        self.current_model, self.current_version = registry.get_production_model(model_id)
        
        # Buffer for new data
        self.data_buffer = []
        self.buffer_size = 1000
        
        # Retraining triggers
        self.retrain_threshold = 500  # New samples
        self.performance_threshold = 0.10  # 10% degradation triggers retrain
    
    def ingest_trace(self, trace: TrainingExample):
        """Ingest new trace and potentially trigger retraining."""
        self.data_buffer.append(trace)
        
        # Check if we should evaluate
        if len(self.data_buffer) >= self.retrain_threshold:
            self._evaluate_and_maybe_retrain()
    
    def _evaluate_and_maybe_retrain(self):
        """Evaluate current model on new data, retrain if needed."""
        
        # Evaluate on recent data
        predictions = self.current_model.predict(self.data_buffer)
        targets = np.array([ex.total_latency_ms for ex in self.data_buffer])
        
        current_mape = np.mean(np.abs(predictions - targets) / targets)
        baseline_mape = self.current_version.metrics['mape'] / 100
        
        degradation = (current_mape - baseline_mape) / baseline_mape
        
        if degradation > self.performance_threshold:
            print(f"Performance degraded by {degradation:.1%}, triggering retrain")
            self._retrain()
        else:
            print(f"Performance stable (degradation: {degradation:.1%})")
            # Add to training data for future
            self._add_to_training_data(self.data_buffer)
        
        # Clear buffer
        self.data_buffer = []
    
    def _retrain(self):
        """Retrain model with new data."""
        
        # Load all training data + new data
        all_data = self._load_all_training_data()
        all_data.extend(self.data_buffer)
        
        # Split
        train, val, test = self._split_data(all_data)
        
        # Train new model
        trainer = TrainingPipeline(TrainingConfig(
            model_type=self.current_version.training_config['model_type'],
            model_config=self.current_version.training_config['model_config']
        ))
        
        new_model, metrics = trainer.run()
        
        # Register new version
        new_version = self.registry.register_model(
            model_id=self.model_id,
            model=new_model,
            training_config=trainer.config.__dict__,
            metrics=metrics,
            description=f"Auto-retrained due to {len(self.data_buffer)} new samples"
        )
        
        # Compare and maybe promote
        if metrics['mape'] < self.current_version.metrics['mape']:
            print(f"New model is better ({metrics['mape']:.2f}% vs {self.current_version.metrics['mape']:.2f}%)")
            self.registry.promote_to_production(self.model_id, new_version.version)
            self.current_model = new_model
            self.current_version = new_version
        else:
            print("New model not better, keeping current production model")
```

---

## Summary

The ML Training Framework provides:

1. **Data Pipeline**: Ingest traces → Extract features → Create training examples
2. **Model Architectures**: XGBoost (fast, interpretable), GNN (graph-aware), Ensemble (best accuracy)
3. **Training Infrastructure**: Scalable training with MLflow tracking
4. **Model Registry**: Version control for trained models
5. **Inference API**: Python SDK and REST API for predictions
6. **Continuous Learning**: Auto-retraining as new data arrives

This enables the architecture exploration platform to make accurate predictions about performance on new hardware configurations.
