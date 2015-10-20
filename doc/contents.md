# OSP director

## Introduction

In short, **OSP director** is Red Hat's new installation, configuration, and monitoring toolset for RHEL OSP deployments. OSP director is a convergence of years of upstream engineering work, established tooling created inhouse and tools that came to us via acquisition to create a best of breed deployment tool that's inline with the overall direction of the OpenStack project. We've also packed many years worth of experience and expertise in deploying OpenStack at scale and with best practices into this new offering, ensuring that it will stand the test of time. It replaces existing tools such as the RHEL OSP installer as the **default** deployment tool as shipped with our OpenStack distribution.

The OSP director initiative came from *three* distinct desires-

1. The need to **remove** the '*bag of installer*' problem; having one comprehensive, flexible, stable, and supportable deployment tool that caters for the vast majority of customer configurations and requirements. Conveying customer confidence and field ability to execute.<br><br>
2. To (finally) onboard **upstream** deployment components as they become more *mature*, utilising the strength of community development and tight **integration** with the rest of the OpenStack components. Eliminating the need for us to work on our own *homebrew* tools - with a limited budget for engineering and quality assurance.<br><br>
3. To have a tool that went **beyond** simply installing an OpenStack environment; previous tools would be 'fire and forget' - once it's installed, the tool has done it's job. But what about ongoing **operational** tasks? Upgrades, patching, monitoring of the environment, capacity planning, utilisation metrics, etc. We needed more than just an installation tool, we needed something that would set us apart from our *competitors*.

OSP director has been designed as a much more **comprehensive** offering, being able to satisfy the requirements of the operations team(s) as well as providing a functional **API**-driven interface for configuration, automation, and eventual deployment. Taking the best features and concepts from the existing tools Red Hat has at its disposal from both community and homebrew solutions, into a **converged** deployment:

<center>
    <img src=./images/installer_roadmap.png>
</center>

