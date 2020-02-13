# Cryogenic Computer Architecture Modeling

----------

## Overview

In this site, we introduce and release CryoCPU,  a cryogenic processor modeling framework. This SW package allows CPU architects to predict the performance of various CPU designs running at the extremely low temperature (i.e., 77K). CryoCPU is built on the top of CryoRAM [\[1\]](#ref_CryoRAM) and CryoCache [\[2\]](#ref_CryoCache) model. 

CryoCPU consists of the following three models: *cryo-MOSFET*, *cryo-wire*, and *cryo-pipeline*.
Cryo-MOSFET and cryo-wire predict MOSFET (i.e., Ion, Ioff) and wire characteristics (i.e., wire resistance) at low-temperatures, respectively. By taking the low-temperature MOSFET and wire characteristics, cryo-pipeline calculates the critical-path delay of each pipeline stage for a given processor design. By utilizing CryoCPU, architects can evaluate the frequency speed-up and the performance gain of the processors running at 77K.

<p align="center">
<img src="https://user-images.githubusercontent.com/60994960/74505014-579f2880-4f39-11ea-8a94-afbf7dd7bd13.png" alt="CryoCPU overview"><br/>
Fig. 1: CryoCPU overview
</p>


### Cryo-MOSFET

**Importance**
MOSFET is the most important basic block of modern computers. Furthermore, the power-related benefits of cryogenic computing mainly come from MOSFET properties (i.e., eliminated subthreshold current, Vdd and Vth scaling). Therefore, modeling the characteristics of MOSFET is crucial for cryogenic computing.

**Limitation in existing models**
BSIM4 [\[3\]](#ref_BSIM4) is a widely-used MOSFET model which predicts the major MOSFET characteristics (i.e., Ion, Ioff) based on the input MOSFET-fabrication information (e.g., doping concentration, gate dielectric thickness). However, BSIM4 does not guarantee its robustness below 200K due to its simple temperature model.

**How we build the model**
We start from BSIM4 as the baseline, and extend its temperature range to 77K as follows.
First, we implement the temperature model for the temperature-sensitive major MOSFET variables (i.e., ueff, vsat, Vth, Rpar). MOSFET variables are parameters determining the MOSFET characteristics (i.e., Ion, Ioff). Among the MOSFET variables, we find that only a few variables (i.e., ueff, vsat, Vth, Rpar) highly depend on the temperature. Based on the insight, we build the temperature-dependency model for the selected variables based on the previous literature.
Second, to support smaller technology nodes, we separately model the above temperature dependency for each gate length. We extract the gate-length dependencies from the MOSFET model card provided from the industry. For more details, please refer to the CryoCore paper.


### Cryo-wire

**Importance**
A metal wire is another critical building block of computers. Furthermore, performance-related benefits of cryogenic computing mainly come from wire properties (i.e., reduced wire resistivity and latency). Therefore, modeling the wire characteristics is crucial for cryogenic computing.

**Limitation in existing models**
There exists a simple model for bulk wire metal, which estimates the wire resistivity for various temperatures.
However, the naive model is only valid for a thick global wire because the model does not consider surface scattering and grain boundary effects. Therefore, we need a more elaborated resistivity model for thin wires in cryogenic processors.

**How we build the model**
We implement cryo-wire, which consists of physics-based equations [\[4\]](#ref_WireModel1), [\[5\]](#ref_WireModel2) including the surface scattering and grain boundary effects. As the effects are determined by the wire's physical geometry (i.e., the width and height), cryo-wire takes the physical dimension of the wire and the target temperature as inputs, and calculates the wire resistivity. Please refer to the CryoCore paper for more modeling details.

### Cryo-pipeline

**Importance**
To evaluate the peak frequency and performance of cryogenic processors, architects should know the critical-path delay of each pipeline stage and how they are affected by the temperature reduction. Therefore, modeling the critical-path delay and its temperature dependence is crucial for cryogenic computing.

**Limitation in existing models**
Previous works [\[6\]](#ref_Complexity),  [\[7\]](#ref_McPAT) built the delay models for major pipeline stages. However, the delay models are only applicable to the room-temperature operation because they do not have the temperature-dependency model.

**How we build the model**
We implement cryo-pipeline, the temperature-aware critical-path delay model, by utilizing Synopsys Design Compiler Topographical Mode [\[8\]](#ref_Synopsys) and our MOSFET/wire model. Cryo-pipeline predicts the critical-path delay as follows.
First, Design Compiler in cryo-pipeline synthesizes a processor layout by utilizing input processor design (Verilog code) and 300K logical/physical libraries. We use BOOM processor design [\[9\]](#ref_BOOM) as the input Verilog code, and utilize FreePDK 45nm [\[10\]](#ref_FreePDK) for the input logical/physical libraries. Second, with the processor layout, Design Compiler calculates the critical-path delay at 300K.
Third, Design Compiler derives the delays at 77K with the same layout by using the 77K libraries generated by cryo-MOSFET and cryo-wire. Finally, cryo-pipeline calculated the frequency speed-up by comparing the critical-path delay at 300K and 77K.
Please refer to the CryoCore paper for more details.

## Usage

### Cryo-MOSFET

#### command:

```shell
./cryo-mosfet {mode} {temperature} {node} {vdd} {vth}
```

#### options:

-   mode: the operation mode for cryo-mosfet:
    -   1: same device in both temperatures (Vth= value of 300K)
    -   2: different device in different temperatures (Vth= value of target temperature + does not consider scaling method)
    -   3: different device in different temperatures (Vth= value of target temperature + scaled with TOX scaling)
    -   4: same device in both temperatures (Vth= value of 300K, DRAM access transistor mode)
    -   5: different device in different temperatures (Vth= value of target temperature, DRAM access transistor mode)
-   temperature: target temperature, in K (77 to 300)
-   node: transistor technology node, in nm
-   vdd: supply voltage, in V
-   vth: target threshold voltage, in V (=Vth0)

#### outputs:

-   Ion: on-channel current per gate width, in A/nm
-   Ileak: leakage current per gate width, in A/nm

### Cryo-wire

#### command:
```shell
./cryo-wire {width} {height} {temperature}
```

#### options:
-   width, height: the width and height of the wire, respectivley, in nm
-   temperature: target temperature, in K (77 to 300)

#### output:
-   ρ: wire resistivity, in Ω * nm

### Cryo-pipeline

#### command:

```shell
./library-converter {cryo-mosfet} {cryo-wire} {logical-library} {physical-library} {temperature} {vdd} {vth} {output-directory}
```

#### options:
-   cryo-mosfet: path to cryo-mosfet binary
-   cryo-wire: path to cryo-wire binary
-   logical-library: path to 300K logical library directory
-   physical-library: path to 300K physical library directory
-   temperature: target temperature, in K (77 to 300)
-   vdd: supply voltage, in V
-   vth: target threshold voltage, in V (=Vth0)
-   output-directory: path for output logical and physical libraries

#### outputs:
-   {output-directory}/logical-library: logical library for the target temperature
-   {output-directory}/physical-library: physical library for the target temperature

## Example use case

The following python script generates logical and physical libraries for the given range of temperatures. The outputs of the script can be utilized by Synopsys design compiler to predict the performance of CPUs at various temperatures.

```python
def sweep_temperature(v_dd, v_th, temp_range):
    for temp in temp_range:
        cmd = [library_converter, cryo_mosfet, cryo_wire,
               logical_library_path, physical_library_path,
               str(temp), str(v_dd), str(v_th), "out/" + str(temp)]

        subprocess.run(cmd)
```

## TODO

We are now finalizing our tools before releasing to the public due to on-going publications (i.e., CryoCore) and result validation process with the industry partners. 


1.  **Refining the interface**
    We are designing a high-level interface of our models for computer architects. It will include 1) glue codes for other major high-level modeling applications, such as CACTI, and 2) Scripts for common use cases, such as finding the optimal input voltages for the given devices.

2.  **Add more data points to the model**
    We are continuously looking for more MOSFET and wire samples to measure and integrating the data to make our model more precise. Currently, we tape-out MOSFET/wire samples with Samsung 28nm process and now wait for the fab-out.

<p align="center">
<img src="https://user-images.githubusercontent.com/60994960/74397116-50075300-4e57-11ea-98a7-7ca789bb6244.png" alt="The die photo of MOSFET and wire samples we designed"><br/>
Fig. 2: The layout of MOSFET and wire samples with Samsung 28nm process
</p>

## Acknowledgements & Validation 

Our tools have been carefully validated with the industry-provided 2x-nm MOSFET data at various temperatures. We are currently adding more validation points. We will update this section after the current phase of data addition and validation are completed.

## Contributors

*(The information is currently hidden for the double-blind review process of on-going publication effort.)*

## References

<a name="ref_CryoRAM">\[1\]</a> G.-h. Lee, D. Min, I. Byun, and J. Kim, “Cryogenic computer architecture modeling with memory-side case studies,” in *Proceedings of the 46th International Symposium on Computer Architecture,* ser. ISCA ’19. New York, NY, USA: ACM, 2019, pp. 774–787. [Online]. Available: http://doi.acm.org/10.1145/3307650.3322219<br/>
<a name="ref_CryoCache">\[2\]</a> D. Min, I. Byun, G.-h. Lee, S.-m. Na, and J. Kim, “Cryocache: A fast, large, and cost-effective cache architecture for cryogenic computing,” in *Proceedings of the Twenty-Fifth International Conference on Architectural Support for Programming Languages and Operating Systems.* ACM, 2020, **to appear,** Accessed: 2019-11-27.<br/>
<a name="ref_BSIM4">\[3\]</a> X. Xi, M. Dunga, J. He, W. Liu, K. M. Cao, X. Jin, J. J. Ou, M. Chan, A. M. Niknejad, C. Hu et al., “Bsim4. 3.0 mosfet model,” *Dept. Elect.  Eng. Comput. Sci., Univ. California, Berkeley, CA, Tech. Rep,* vol. 94720, p. 30, 2003.<br/>
<a name="ref_WireModel1">\[4\]</a> C.-K. Hu, J. Kelly, J. H. Chen, H. Huang, Y. Ostrovski, R. Patlolla, B. Peethala, P. Adusumilli, T. Spooner, L. Gignac et al., “Electromigration and resistivity in on-chip cu, co and ru damascene nanowires,” in *2017 IEEE International Interconnect Technology Conference (IITC).* IEEE, 2017, pp. 1–3.<br/>
<a name="ref_WireModel2">\[5\]</a> C.-K. Hu, J. Kelly, H. Huang, K. Motoyama, H. Shobha, Y. Ostrovski, J. H. Chen, R. Patlolla, B. Peethala, P. Adusumilli et al., “Future onchip interconnect metallization and electromigration,” in *2018 IEEE International Reliability Physics Symposium (IRPS).* IEEE, 2018, pp. 4F–1.<br/>
<a name="ref_Complexity">\[6\]</a> S. Palacharla, N. P. Jouppi, and J. E. Smith, *Complexity-effective superscalar processors.* ACM, 1997, vol. 25, no. 2.<br/>
<a name="ref_McPAT">\[7\]</a> S. Li, J. H. Ahn, R. D. Strong, J. B. Brockman, D. M. Tullsen, and N. P. Jouppi, “Mcpat: an integrated power, area, and timing modeling framework for multicore and manycore architectures,” in *Proceedings of the 42nd Annual IEEE/ACM International Symposium on Microarchitecture.* ACM, 2009, pp. 469–480.<br/>
<a name="ref_Synopsys">\[8\]</a> Synopsys, Synopsys dc-ultra, 2019. [Online]. Available: https://www.synopsys.com/implementation-and-signoff/rtl-synthesis-test/dc-ultra.html<br/>
<a name="ref_BOOM">\[9\]</a> C. Celio, D. A. Patterson, and K. Asanovic, “The berkeley out-of-order machine (boom): An industry-competitive, synthesizable, parameterized risc-v processor,” *EECS Department, University of California, Berkeley, Tech. Rep. UCB/EECS-2015-167,* 2015.<br/>
<a name="ref_FreePDK">\[10\]</a> J. E. Stine, I. Castellanos, M. Wood, J. Henson, F. Love, W. R. Davis, P. D. Franzon, M. Bucher, S. Basavarajaiah, J. Oh et al., “Freepdk: An open-source variation-aware design kit,” in *2007 IEEE international conference on Microelectronic Systems Education (MSE’07).* IEEE, 2007, pp. 173–174.<br/>
