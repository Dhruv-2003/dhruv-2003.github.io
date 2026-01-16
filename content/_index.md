+++
template = 'home.html'

[extra]
lang = 'en'

# Show footer in home page
footer = true

# If you don't want to display id/bio/avatar, simply comment out that line
name = "Dhruv Agarwal"
id = "0xdhruv"
bio = "software engineer & blockchain researcher"
avatar = "img/avatar.webp"
links = [
    { name = "GitHub", icon = "github", url = "https://github.com/Dhruv-2003" },
    { name = "LinkedIn", icon = "linkedin", url = "https://linkedin.com/in/dhruvagarwalofficial" },
    { name = "Twitter", icon = "twitter", url = "https://twitter.com/0xdhruva" },
    { name = "Email", icon = "email", url = "mailto:contact.dhruvagarwal@gmail.com" },
]

# Show a few recent posts in home page
recent = true
recent_max = 15
recent_more_text = "more »"
date_format = "%b %-d, %Y"
+++

Hi, I'm Dhruv, an engineer passionate about research in blockchain, cryptography, and decentralized systems. My interests include Zero Knowledge, Modular Blockchain, and DeFi.

Currently, I'm working as an Engineer at [Conduit](https://conduit.xyz), building rollup infrastructure. Alongside, I'm experimenting & building systems, deepening my understanding of protocols and core Ethereum architecture. As a three-time ETHGlobal finalist and a frequent hackathon participant, I'm always eager to take on new challenges and push the boundaries of what's possible.

I like travelling and photography, and I'm always up for a good conversation about the latest in tech or engineering. Feel free to reach out.

## Publications

{{ collection(file="publications/publications.toml") }}

## Experience

{{ collection(file="experience/experience.toml") }}
