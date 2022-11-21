# Blog

Blog source. The blog uses a Jekyll theme called [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/)

## Building

The blog is built and deployed using Cloud Build. The `cloudbuild.yaml` file drives this.

## Dependencies

Based of these [instructions](https://jekyllrb.com/docs/installation/ubuntu/).

    sudo apt-get install ruby-full build-essential zlib1g-dev
    echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
    echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
    echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    gem install jekyll bundler
    bundle

## Running locally

    bundle exec jekyll serve

**Commments don't work locally** unless `JEKYLL_ENV=production` is set.

## Updating dependencies

Running `bundle update` should update the `Gemfile.lock`.