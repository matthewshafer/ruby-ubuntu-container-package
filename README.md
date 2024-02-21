# Ruby Ubuntu Container Package

Welcome!

I created this project so I didn't have to build ruby in each container for my own projects.
The resulting deb is stored in a scratch image which I can reference and install in a container.

### Notes

Built With:

  * `--prefix=/usr`
  * `--disable-install-doc`

Options That Are Not Set:

 * `--enable-shared`
 * `--with-jemalloc` (I LD_PRELOAD jemalloc when needed)

Architecture:

  * amd64(x86_64)
  * arm64(aarch64)

### 20.04 Dependencies

Runtime:

  * Ruby 2.6-3.0: `libffi7 libgmp10 libreadline8 libssl1.1 libyaml-0-2 zlib1g libgdbm6 libgdbm-compat4`
  * Ruby 3.1+: `libffi7 libgmp10 libreadline8 libssl1.1 libyaml-0-2 zlib1g`

Building Native Extensions:

  * Ruby 2.6-3.0: `libffi-dev libgmp-dev libreadline-dev libssl-dev libyaml-dev zlib1g-dev libgdbm-dev libgdbm-compat-dev`
  * Ruby 3.1+: `libffi-dev libgmp-dev libreadline-dev libssl-dev libyaml-dev zlib1g-dev`

### 22.04 Dependencies

Runtime:

  * Ruby 3.1+: `libffi7 libgmp10 libreadline8 libssl3 libyaml-0-2 zlib1g`

Building Native Extensions:

  * Ruby 3.1+: `libffi-dev libgmp-dev libreadline-dev libssl-dev libyaml-dev zlib1g-dev`

## Contributing

Since this project is for myself I'm not looking to build multiple versions with different build options unless my uses change.
I'm not accepting PR's for ruby version bumps, feel free to open an issue though.
