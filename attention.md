```python
import math

import torch
import torch.nn as nn


class SDPA(nn.Module):

    def forward(self, Q, K, V):
        """
        Q, K, V: (B, T, d_k) / (B, T, d_v)
        """
        d_k = Q.shape(-1)
        # (B, T, d_k) @ (B, d_k, T) -> (B, T, T)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)

        attn = torch.softmax(scores, dim=-1) # (B, T, T)
        output = torch.matmul(attn, V) # (B, T, d_v)
        return output, attn


class MultiHeadAttention(nn.Module):

    def __init__(self, hidden_size, num_heads, d_k=None, d_v=None):
        self.hidden_size = hidden_size
        self.num_heads = num_heads

        self.d_k = d_k or (hidden_size // num_heads)
        self.d_v = d_v or (hidden_size // num_heads)

        self.W_q = nn.Linear(hidden_size, num_heads * d_k)
        self.W_k = nn.Linear(hidden_size, num_heads * d_k)
        self.W_v = nn.Linear(hidden_size, num_heads * d_v)

        self.fc_out = nn.Linear(num_heads * d_v, hidden_size)
        
        self.attn = SDPA()

    def forward(self, x):
        """
        x: (B, T, H)
        """
        B, T, _ = x.size()

        Q = self.W_q(x).view(B, T, self.num_heads, self.d_k).transpose(1, 2) # (B, heads, T, d_k)
        K = self.W_k(x).view(B, T, self.num_heads, self.d_k).transpose(1, 2) # (B, heads, T, d_k)
        V = self.W_v(x).view(B, T, self.num_heads, self.d_v).transpose(1, 2) # (B, heads, T, d_v)

        # (B, heads, T, D) -> (B*heads, ...)
        Q = Q.view(B * self.num_heads, T, self.d_k)
        K = K.view(B * self.num_heads, T, self.d_k)
        V = V.view(B * self.num_heads, T, self.d_v)

        out, attn = self.attn(Q, K, V)
        out = out.view(B, self.num_heads, T, self.d_v).transpose(1, 2) # (B, heads, T, d_v) -> (B, T, heads, d_v)
        out = out.view(B, T, self.num_heads * self.d_v) # (B, T, heads * d_v)
        out = self.fc_out(out) # (B, T, hidden_size)

        return out, attn
```
