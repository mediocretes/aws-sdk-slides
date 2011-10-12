code-engine: coderay

# Semi-Precious Gems: AWS-SDK

->Look guys, I stopped using keynote<-

->Nathan D Acuff<-

->iGoDigital<-



What's wrong with right_aws?
============================

- Nothing, shut up
- Amazon probably won't break it
- It's not as new as aws-sdk

You don't seem very angry
=========================

- (Almost) The only documentation for aws-sdk is the source code
- They love to rebase
- It is very raw, like EC2 (but powerful, like EC2)


It only does everything
=======================

* Manage Your Instances
* Do Things with your Load Balancer
* Identity Access Management
* Poorly named ORM
* S3
* SimpleDB
* Simple Email Service
* Notifications
* Simple Queue Service
* Key and Token Management

Managing instances
==================

<% code do %>
instances = ec2.images['sweet-ruby-ami'].run_instances(5)
# or
mongo_instances = ec2.instances.create(:image_id => 'mongo-ami', 
    :count => 3, type => "m2.16xlarge"
    :block_device_mappings => { "/dev/sdm" = > { 
        :volume_size => 1000, 
        :delete_on_termination => false}
    "/dev/sdn" => {
        :volume_size => 1000,
        :delete_on_termination => false}})
<% end %>

Load Balancing
==============

<% code do %>
balancer = elb.load_balancers.create('balance-you-some-ruby',
    :availability_zones => ["us-east-1a"],
    :listeners => [{
        :port => 80,
        :protocol => :http,
        :instance_port => 80,
        :instance_protocol => :http
    }])
balancer.register instances_from_the_last_slide
balancer.health.inject({}) do |all, inst| 
    all[inst[:instance]] = inst[:state]; all
end
<% end %>

Identity and Access Management
==========================
<% code do %>
nate = users.create('nate', :path => '/staff/cie/nate')
nate.login_profile.password = 'password'
nate.policies["CrashDatabase"] = Policy.new do |p|
    policy.allow(:actions => ['ec2:TerminateInstances'],
        :resources => some_instance,
        :principals => :any).where
#Amazon policies are super complex
#also there are user groups
<% end %>

Object Mapper
=============

->Not really an ORM<-
<% code do %>
class Product < AWS::Record::Base
    string_attr :name
    datetime_attr :release_date
    float_attr :price #yeah, I know
    string_attr :description
    string_Attr :tags, :set => true
    timestamps
end
p = Product.new
p.name = 'Twilight'
p.price = 7.99
p.tags = ['teen', 'paranormal', 'romance']
p.save
<% end %>

More Object Mapper
==================
<% code do %>
class PageView < AWS::Record::Base
   string_attr :url
   string_attr :user_id
   validates_presence_of :url
end

pv = PageView.new
pv.valid?
pv.save # should throw up
<% end %>

Even More Object Mapper
=======================
<% code do %>
p = Product["UUID-which-is-the-product-id"]
p = Product.first("release_date > ?", Time.now)
Product.find do |p|
    do_some_stuff_to p
end
class Product < AWS::Record::Base
    scope :available, where("release_date < ?", Time.now)
end
new_released_books = Product.available.top_10.order(:release_date, :desc)
<% end %>
->Lazy, Optimistic, Not Relational<-

->Notice how I didn't mention joins?<-

S3
===
<% code do %>
bucket = s3.buckets.create("pictures-of-cats")
bucket.objects.each do |obj|
    puts object.key #lazy!
end
kitty = 'pictures/cats/2011/october/week1/kitty0473.jpg'
bucket.objects['kitty.jpg'].write(File.read(kitty))
File.open("kitty.jpg", 'w') do |f|
    f.write(bucket.objects['kitty.jpg'].read)
end
<% end %>
->![kitty](kitty.jpg)<-

Bet You Thought We Were Done With SimpleDB
==========================================

<% code do %>
#If you don't like pretending you have an ORM...
products = sdb.domains.create('books')
item = products.items['twilight']
eds = item.attributes['editions']
eds.add 'hardcover', 'softcover', 'kindle', 'audiobook' #immediate
products.delete['twlight']

#and pretty much everything else you'd expect
<% end %>

Simple Email Service
====================
<% code do %>
config.action_mailer.delivery_method = :amazon_ses

#or, to do things directly
ses.verify('nacuff@igodigital.com')
ses.send_mail(:subject => 'More Cats!')#and the rest of the hash
<% end %>

->Also support for quotas, stats, access control, bounces, complains, and so on<-

Simple Notification Service
===========================

<% code do %>
sns.topics['server_alert'].subscribe('nacuff@igodigital.com')
sns.topics['server_alert'].publish('hey, FYI, the server is down')

suckers = sns.topics['server_alert'].subscriptions.select do |s|
    s.protocol == :email
end  #get all the email subscribed to this
    
sns.topics['admin'].publish("emailed " + suckers.map(&:endpoint))
<% end %>

SQS
===

* Everything is text
* Up to 8k up to 64k
* Messages arrive late and multiple times
* You might want to use threads to write

More SQS
========

<% code do %>
queue = sqs.queues.create('add_cats_to_images')
message = { :s3_location => 'your_details_here',
    :s3_cat_location => 'your_cat_here',
    :output_location => 'your_filename_here'
    :message_sent => Time.new}
queue.send_message message.to_yaml

#or if you care about speed
Thread.new { queue.send_message... }
<% end %>

Receiving SQS Messages
======================

<% code do %>
queue.poll{ |message| ... }
daemonization_loop.do{ queue.get{ |message| ... }}
#there are other techniques, but this one manages all your details
<% end %>

* Messages are probably locked immediately
* Messages automatically unlock after a delay (5 minutes?)
* Queue size is a magical, capricious animal

Let's talk Idempotency
======================

-> f(x) = f(f(x)) <-

<% code do %>
queue.get do |message|
    #bad
    db.items[message[:id]].counter += 1
    #okay
    db.items[:message_counts].create(message[:unique_id])
    db.items[message[:item_id]].counter = db.items[:message_counts].count
    #If you need to do a lot of work...
    output = images[message[:output_location]]
    return if output.updated_since message[:message_sent]
    img = images[message[:s3_location]]
    cats = images[message[:s3_cat_location]]
    return if output.updated_since message[:message_sent]
    img.apply_cats cats
    return if output.updated_since message[:message_sent]
    output.save    
    #DO NOT WRITE TO YOUR INPUT
end
<% end %>

Key and Token Management
==========================

<% code do %>
#this will last forever
s3 = AWS::S3.new(:access_key_id => 'your_real_key', 
    :secret_access_key => 'your_other_real_key')
    
#this will not
token = AWS::STS.new
session = token.new_session
s3 = AWS::S3.new(session.credentials)  
<% end %>

It is time to drink
===================

* Holy crap that was a marathon
* This gem seems huge
* I only skimmed over a lot of the features
* We are only using the SQS portion in production (so far)

-> ![more_cat](kitty2.jpg) <-
