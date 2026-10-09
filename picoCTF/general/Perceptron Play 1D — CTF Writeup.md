# Perceptron Play 1D — CTF Writeup

## Challenge Overview

| Field | Value |
|-------|-------|
| **Name** | Perceptron Play 1D Alpha |
| **Category** | Artificial Intelligence / AI Foundations I |
| **Difficulty** | Easy |
| **Author** | LT 'syreal' Jones |
| **Connection** | `nc xebec.cylabacademy.net 29506` |

The challenge is an interactive playground where you must tune the parameters of a single-layer perceptron operating on a 1-dimensional number line. The goal is to correctly classify every labeled training point. Once all points are classified correctly, the `CHECK` command reveals the flag.

---

## Background: The 1D Perceptron

A single-layer perceptron computes a weighted sum of its inputs plus a bias:
