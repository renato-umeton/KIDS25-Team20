# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains WDL (Workflow Description Language) 1.2 workflows for genomic variant calling using Nvidia Clara Parabricks v4.5.1. These workflows are GPU-accelerated bioinformatics pipelines designed to run on systems with NVIDIA GPUs.

The workflows were adapted from the Clara Parabricks WDL repository and validated using Sprocket 0.17.1 as the workflow execution engine.

## Repository Structure

```
.
├── workflows/       # High-level workflow definitions
│   ├── fq2bam.wdl
│   ├── bam2fq2bam.wdl
│   ├── germline_calling.wdl
│   ├── somatic_variant.wdl
│   └── deepvariant.wdl
└── tasks/          # Task implementations (called by workflows)
    ├── fq2bam.wdl
    ├── bam2fq2bam.wdl
    ├── germline_calling.wdl
    ├── somatic_variant.wdl
    └── deepvariant.wdl
```

## Workflow Architecture

All workflows follow a two-tier architecture:

1. **Workflow layer** (`workflows/`): Defines inputs, parameter metadata, and orchestrates task calls
2. **Task layer** (`tasks/`): Contains the actual compute tasks with container requirements and command execution

Workflows import their corresponding task definitions using relative imports:
```wdl
import "../tasks/fq2bam.wdl" as tasks
```

## Available Workflows

1. **fq2bam**: Align FASTQ reads to BAM using BWA-mem with optional BQSR
2. **bam2fq2bam**: Extract FASTQ from BAM and realign to a different reference
3. **germline_calling**: Run HaplotypeCaller and/or DeepVariant for germline variants
4. **somatic_variant**: Run Mutect2 for tumor-normal somatic variant calling
5. **deepvariant**: Standalone DeepVariant variant calling with retraining capabilities

## Container and Runtime Requirements

All tasks use the Clara Parabricks container:
```
nvcr.io/nvidia/clara/clara-parabricks:4.5.1-1
```

Key runtime requirements in task definitions:
- `gpu: true` - All tasks require GPU access
- `container:` - Specifies the Docker/Singularity image
- `cpu:`, `memory:`, `disks:` - Resource requirements
- Tasks in `tasks/` directory contain `requirements` and `hints` blocks

The deepvariant standalone workflow uses:
```
google/deepvariant:1.9.0-gpu
```

## Working with WDL Files

### Task Structure
Tasks define:
- `input {}`: Required and optional parameters with defaults
- `command <<<>>>`: Bash commands executed in container
- `output {}`: Files and values produced
- `requirements {}`: Resource and container specs
- `hints {}`: Additional execution hints (gpuCount, gpuType, etc.)

### Workflow Structure
Workflows define:
- `meta {}`: Description, author, and output documentation
- `parameter_meta {}`: Input parameter descriptions
- `input {}`: Workflow-level inputs
- Task calls with input mapping
- Conditional execution using `if` blocks
- `output {}`: Final workflow outputs

### Common Patterns

**Conditional execution:**
```wdl
if (run_deep_variant) {
    call germline_calling.deepvariant { ... }
}
```

**Reference tarball handling:**
All tasks expect reference genomes as tar archives and extract them:
```bash
tar xvf ~{input_ref_tarball}
```

**File naming conventions:**
- Output base names derived from input: `basename(input_bam, ".bam")`
- Parabricks outputs use `.pb.` prefix: `~{outbase}.pb.bam`

**Read group formatting:**
```bash
"@RG\tID:~{rg_id}\tLB:~{read_group_library_name}\tPL:~{read_group_platform_name}\tSM:~{read_group_sample_name}"
```

## Key Technical Details

### Parabricks Commands

Tasks use `pbrun` commands:
- `pbrun fq2bam`: FASTQ to BAM alignment with BWA-mem
- `pbrun haplotypecaller`: GPU-accelerated GATK HaplotypeCaller
- `pbrun deepvariant`: GPU-accelerated DeepVariant
- `pbrun mutect2`: GPU-accelerated Mutect2 somatic calling

### GATK Best Practices Support

The `germline_calling` workflow supports GATK best practices with:
- Quantization bands: `-GQB 10 -GQB 20 -GQB 30...`
- Static quantized quals
- Standard annotations for gVCF mode

### Optional Features

- **BQSR (Base Quality Score Recalibration)**: Enabled via `known_sites_vcf` parameter
- **gVCF mode**: Available in germline calling workflows
- **Panel of Normals**: Optional filtering in somatic workflow
- **Adaptive pruning**: Mutect2 pruning option in HaplotypeCaller

## Development Workflow

When modifying workflows:

1. Maintain WDL 1.2 version specification at top of files
2. Keep workflow and task files in sync (same base name in `workflows/` and `tasks/`)
3. Update `meta` and `parameter_meta` blocks when changing inputs/outputs
4. Preserve copyright headers where present
5. Test changes with Sprocket before committing

### Adding New Parameters

When adding optional parameters:
1. Add to workflow `input {}` with default value or `?` for optional
2. Add to `parameter_meta {}` with description
3. Pass through to task call
4. Add to task `input {}`
5. Use in command with conditional syntax: `~{if condition then "--flag " else ""}`

### Resource Sizing

Tasks compute disk space automatically when `disk_gb = 0`:
```wdl
Int auto_disk_gb = if disk_gb == 0 then
    ceil(2.0 * size(input_bam, "GB")) + ...
    else disk_gb
```

## Common Issues

1. **GPU availability**: All tasks require GPU; ensure runtime environment has NVIDIA GPU support
2. **Reference format**: References must be provided as tar archives, not individual files
3. **Index files**: BAM index (.bai) and VCF index (.tbi) files must accompany primary files
4. **Container access**: Ensure access to nvcr.io/nvidia containers

## Execution with Sprocket

This repository is validated with Sprocket 0.17.1. While specific Sprocket commands are not documented in the repo, standard WDL execution patterns apply with JSON input files.
