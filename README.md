# Linux namespaces

This repo contains the material of our Linux namespaces mid-point presentation.

## Introduction

This project provides an overview and demonstration of Linux namespaces. Linux namespaces are a feature of the Linux kernel that partitions kernel resources so that one set of processes sees one set of resources while another set of processes sees a different set of resources.

We focused the mount namespace which is a feature in the Linux kernel that allows for the isolation of mount points between different sets of processes. This means that changes to the filesystem layout (such as mounting or unmounting filesystems) in one namespace do not affect other namespaces.

## Table of Contents

1. [Presentation](Presentation.pptx)
2. [Video](Linux_Namespaces.mp4)
3. [Demo](demo.md)
4. [Kernel code browsing](the_mount_namespace.md)

## Setup and Running

No specific setup or running instructions are provided for this project, just a linux machine :)
