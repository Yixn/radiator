require 'bundler/gem_tasks'
require 'rake/testtask'
require 'yard'
require 'radiator'
require 'awesome_print'

Rake::TestTask.new(:test) do |t|
  t.libs << 'test'
  t.libs << 'lib'
  t.test_files = FileList['test/**/*_test.rb']
  t.ruby_opts << if ENV['HELL_ENABLED']
    '-W2'
  else
    '-W1'
  end
end

YARD::Rake::YardocTask.new do |t|
  t.files = ['lib/**/*.rb']
end

task default: :test

namespace :clean do
  desc 'Deletes test/fixtures/vcr_cassettes/*.yml so they can be rebuilt fresh.'
  task :vcr do |t|
    exec 'rm -v test/fixtures/vcr_cassettes/*.yml'
  end
end

desc 'Tests the ability to broadcast live data.  This task broadcasts a claim_reward_balance of 0.0000001 VESTS.'
task :test_live_broadcast, [:account, :wif, :chain] do |t, args|
  account_name = args[:account] || 'social'
  posting_wif = args[:wif] || '5JrvPrQeBBvCRdjv29iDvkwn3EQYZ9jqfAHzrCyUvfbEbRkrYFC'
  chain = (args[:chain] || 'steem').to_sym
  # url = 'https://testnet.steemitdev.com/' # use testnet
  url = nil # use default
  options = {chain: chain, wif: posting_wif, url: url}
  tx = Radiator::Transaction.new(options)
  
  reward_core, reward_debt, reward_vest = case chain
  when :steem then ['0.000 STEEM', '0.000 SBD', '0.000001 VESTS']
  when :hive then ['0.000 HIVE', '0.000 HBD', '0.000001 VESTS']
  end
  
  case chain
  when :steem
    tx.operations << {
      type: :claim_reward_balance,
      account: account_name,
      reward_steem: reward_core,
      reward_sbd: reward_debt,
      reward_vests: reward_vest
    }
  when :hive
    tx.operations << {
      type: :claim_reward_balance,
      account: account_name,
      reward_hive: reward_core,
      reward_hbd: reward_debt,
      reward_vests: reward_vest
    }
  end
  
  response = tx.process(true)
  ap response
  
  if !!response.result
    result = response.result
    
    case chain
    when :steem
      puts "https://steemd.com/b/#{result[:block_num]}" if !!result[:block_num]
      puts "https://steemd.com/tx/#{result[:id]}" if !!result[:id]
    when :hive
      puts "https://hiveblocks.com/b/#{result[:block_num]}" if !!result[:block_num]
      puts "https://hiveblocks.com/tx/#{result[:id]}" if !!result[:id]
    else
      puts result
    end
  end
end

desc 'Tests the ability to stream live data. defaults: chain = steem; persist = true.'
task :test_live_stream, [:chain, :persist] do |t, args|
  chain = (args[:chain] || 'hive').to_sym
  persist = (args[:persist] || 'true') == 'true'
  last_block_number = 0
  # url = 'https://testnet.steemitdev.com/'
  url = chain == :steem ? 'https://api.steemit.com' : 'http://anyx.io'
  # url = nil # use default
  options = {chain: chain, persist: persist, url: url}
  total_ops = 0.0
  total_vops = 0.0
  elapsed = 0
  count = 0
  
  Radiator::Stream.new(options).blocks do |b, n, api|
    start = Time.now.utc
    
    if last_block_number == 0
      # skip test
    elsif last_block_number + 1 == n
      t = b.transactions
      t_size = t.size
      o = t.map(&:operations)
      op_size = o.map(&:size).reduce(0, :+)
      total_ops += op_size
      
      catch :try_vops do; api.get_ops_in_block(n, true) do |vops, error|
        if !!error
          puts "Error on get_ops_in_block for block #{n}"
          ap error if defined? ap
        end
        
        if vops.nil?
          puts "#{n}: #{b.witness}; Problem: vops is nil!  Retrying ..."
          sleep 3 # Possibly fall behind a bit and catch up later.
          throw :try_vops
          
          # Did we reach this point with an unhandled error that wasn't retried?
          # If so, vops might be nil and we might need this error to get handled
          # instead of checking for vops.nil?.
        end
      
        vop_size = vops.size
        total_vops += vop_size
        
        vop_ratio = if total_vops > 0
          total_vops / total_ops
        else
          0
        end
        
        elapsed += Time.now.utc - start
        count += 1
        puts "#{n}: #{b.witness}; trx: #{t_size}; op: #{op_size}, vop: #{vop_size} (cumulative vop ratio: #{('%.2f' % (vop_ratio * 100))} %; average #{((elapsed / count) * 1000).to_i}ms)"
      end; end
    else
      # This should not happen.  If it does, there's likely a bug in Radiator.
      
      puts "Error, last block number was #{last_block_number}, did not expect #{n}."
    end
    
    last_block_number = n
  end
end

desc 'Ruby console with radiator already required.'
task :console do
  exec "irb -r radiator -I ./lib"
end

desc 'Build a new version of the radiator gem.'
task :build do
  exec 'gem build radiator.gemspec'
end

desc 'Publish the current version of the radiator gem.'
task :push do
  exec "gem push radiator-#{Radiator::VERSION}.gem"
end

# We're not going to yank on a regular basis, but this is how it's done if you
# really want a task for that for some reason.

# desc 'Yank the current version of the radiator gem.'
# task :yank do
#   exec "gem yank radiator -v #{Radiator::VERSION}"
# end
