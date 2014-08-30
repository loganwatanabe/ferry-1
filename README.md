# Ferry

Ferry is a data migration and data manipulation tool that seeks to quickly and easily reduce overhead when dealing with big data problems.

## TO-DO

- [ ] Refactoring before public release
  - [x] Define action-items for refactor
  - [x] Provide working example(s) of using ferry
  - [ ] Public release fine-tuning
- [ ] Tests
  - [ ] Testing input for migrate method (max_workers, batch_size)
  - [ ] Testing that there is an ActiveRecord::Relation object being passed to find_in_batches
  - [ ] Migration Scenarios - dummy class migration
- [ ] Refactor logging logic into Logger class
  - [x] Initial revision
  - [ ] Review

## Installation

Add this line to your application's Gemfile:

    gem 'ferry'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ferry

## Usage

Usage pending. See examples / submit PR's for your ideas.

## Example(s)

###### 29 July 2014
Version 0.0.1 is functional with the rake task defined here :: https://github.com/customink/design_content_migration/blob/master/lib/tasks/ferry_example.rake#L10

Please manually install ferry from your locally cloned repo ...
```
git clone git@github.com:customink/ferry.git
cd ferry
gem build ferry.gemspec
gem install ferry
```
add it to your app's Gemfile
```
gem 'ferry'
```
and then
```
bundle install
```
as it has not been pushed to rubygems.com yet.

Tests - Coming soon to an editor near me!

###### 28 July 2014
Ferry should not support Oracle.

###### 25 July 2014
After a few more reviews with @metaskills, @gilr00y, @jdlehman, and @danielwheeler1987, Ferry will extend ActiveRecord with a "migrate" (more legit name search still in naming progress) method. From there we are going to pass the same relation to find in batches to a worker which will plow through the batch passed to it via a yield call from the task.

Tests will include; validate the data passed into the worker (log) and testing that there is an ActiveRecord::Relation being passed to find_in_batches.

###### 23 July 2014
After a few chats with @gilr00y and @jdlehman Ferry may extend ActiveRecord with a "migrate" method we could call on an ActiveRecord object. From there that object would call an Engine instance with appropriate fields to kickoff the actual data migration.

There is some logic duplication and layer duplication between the Engine class and the "migrate" method that extends ActiveRecord. Still working out how to concisely write logic that handles the management of forking connection and engine init calls.

```
require "ferry/version"
require 'models/engine'
require 'models/logger'

module Ferry
  class ActiveRecord
    def self.migrate(&block)
      yield
    end
  end
end
```

This implementation should be able to run something like this ...

```
engine = Engine.new(
  Design.where("savedate > ?", 6.months.ago.strftime("%d.%m.%Y %H").to_datetime).id,
  Design.where("savedate > ?", 3.months.ago.strftime("%d.%m.%Y %H").to_datetime).id,
  100_000,
  1_000,
  "log/ferry"
)

Design.where("savedate > ?", 130.hours.ago.strftime("%d.%m.%Y %H").to_datetime).migrate(
  engine.run do | start_id, end_id, chunk_size, batch_size, log |
    worker.run do | start_id, chunk_size, batch_size, log |
      worker_end_id = start_id + chunk_size - 1
      Design.where("id >= ? && id <= ?", start_id, worker_end_id).find_in_batches(batch_size: batch_size) do |batch|
        # move and manipulate data as you please
      end
      start_id += batch_size
    end
  end
)

```

###### 22 July 2014
After installing ferry to your local machine or bundling from your gemfile - in your migration task make sure to define your chunker as such ...

```
require 'ferry'

namespace :example do
  task "my_migration_task" do

    ferry = Engine.new(
      :max_workers  => number_of_workers ex:8,
      :start_id     => where_are_we_starting ex:2910, Model.first.id,
      :end_id       => where_are_we_ending ex:8190, Model.last.id,
      :chunk_size   => size_of_chunks_that_workers_will_process ex:42,
      :working_dir  => ex:"path/to/working_dir"
    )

    ferry.run do |start_id, chunk_size, log|
      begin
        work = Model.select(":id").where("? <= id and id < ?", start_id, start_id + chunk_size)
        rows_to_process = rel.count
        log.puts("rows_to_process: #{rows_to_process}")
        work.find_in_batches(:batch_size => 1_000) do
          # doing things and logging stuff as you please ...
        end
      rescue Exception => e
        log.puts "Broken on id #{id}"
        raise e
      end
    end

  end
end
```


## Contributing

1. Fork it ( https://github.com/[my-github-username]/ferry/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
