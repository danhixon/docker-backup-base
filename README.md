docker-backup-base
==================

Base Dockerfile for Using the Backup Gem inside a docker container. This base currently installs [mongodb tools](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-debian/), [postgres client tools](http://www.postgresql.org/docs/9.3/static/reference-client.html), [gnupg](http://www.gnupg.org), the [backup gem](http://backup.github.io/backup/v4/) and the [whenever gem](https://github.com/javan/whenever).

Use this as your base image, then `ADD` your Backup/config.rb, Backup/models/*, and your schedule.rb. Something like this:

	FROM danhixon/backup-base
    # backup_passphrase contains your gnupg passphrase.
    ADD backup_passphrase /root/backup_passphrase
    RUN mkdir -p /root/Backup/models
    ADD config.rb /root/Backup/config.rb
    ADD models/my_backups.rb /root/Backup/models/my_backups.rb
  
    RUN mkdir -p /root/Whenever/config
    ADD schedule.rb /root/Whenever/config/schedule.rb
    RUN cd /root/Whenever ; whenever --write-crontab
    
After building the file run the container with your S3 credentials and links to the database servers:

    docker run --name backups --link postgres:pg --link mongo:mongodb -e AWS_ACCESS_KEY_ID=#{pg_appuser} -e AWS_SECRET_ACCESS_KEY=#{pg_appuser_password} awesome_app/backup

#Sample Files

###Backup/models/backups.rb
	Model.new(:hourly_backup, 'PostgreSQL and MongoDB Hourly (24)') do
	  database MongoDB
	  database PostgreSQL
	  encrypt_with GPG
	  compress_with Gzip
	  notify_by Mail

	  store_with S3 do |s3|
	    s3.keep  = 24
	    s3.path  = "/db/hourly/"
	  end
	end

	Model.new(:daily_backup, 'PostgreSQL and MongoDB Daily (7)') do
	  database MongoDB
	  database PostgreSQL	  
	  encrypt_with GPG
	  compress_with Gzip
	  notify_by Mail

	  store_with S3 do |s3|
	    s3.keep     = 7
	    s3.path  = "/db/daily/"
	  end
	end

###Backup/config.rb

	Backup::Database::PostgreSQL.defaults do |db|
	  db.database           = "awesome_app"
	  db.username           = ENV['PG_ENV_POSTGRESQL_USER']
	  db.password           = ENV['PG_ENV_POSTGRESQL_PASSWORD']
	  db.host               = ENV['PG_PORT_5432_TCP_ADDR']
	  db.port               = ENV['PG_PORT_5432_TCP_PORT']
	  # When dumping all databases, `skip_tables` and `only_tables` are ignored.
	  #db.skip_tables        = ["skip", "these", "tables"]
	  #db.only_tables        = ["only", "these", "tables"]
	  db.additional_options = ["-Fc", "-E=utf8","--no-acl","--no-owner"]
	end

	Backup::Database::MongoDB.defaults do |db|
	  db.database           = "awesome_app"
	  db.username           = ENV['MONGODB_ENV_USER']
	  db.password           = ENV['MONGODB_ENV_USER_PASSWORD']
	  db.host               = ENV['MONGODB_PORT_27017_TCP_ADDR']
	  db.port               = ENV['MONGODB_PORT_27017_TCP_PORT']
	  db.ipv6               = false
	  #db.only_collections   = ["only", "these", "collections"]
	  db.additional_options = []
	  db.lock               = false
	  db.oplog              = false
	end

	Backup::Storage::S3.defaults do |s3|
	  s3.access_key_id     = ENV["AWS_ACCESS_KEY_ID"]
	  s3.secret_access_key = ENV["AWS_SECRET_ACCESS_KEY"]
	  s3.bucket            = "backups.domain.com"
	end

	Encryptor::GPG.defaults do |gpg|
		# gpg.gpg_homedir = 
		gpg.keys = {}
		gpg.keys['dan.hixon@email.com'] = <<-KEY
	      -----BEGIN PGP PUBLIC KEY BLOCK-----
	Version: GnuPG v1

	mQENBFNQQGMBexJfaytN6xgD90Oyo2a2r+DzuO122fuEN+8R4nh
	N81LXSfVukfU+xAaK95ASLHoLfXAdubTr23mE+XkVNFJJFSnMtcfYvYIJR7StQii
	/q87TNTKwJvX+V5s+OMKWz5V/JZlRY6BJ0bcIR8PzAWRTPbEkluljNhN2p70rF4U
	r4CqZNVQrbTPFyZdN06TJk+5982C5QRnfm27fZkCn8V7PZkzzARZP1uIVoYnqIaA
	Fx38x6HOPhUNnYnpwQH5eE1+RrKHc/1xfjxyxa7FdTE90jJk3HoIPLlSXI2pZNpt
	tpWxYdcHLPP7aaSPjc+mrPTcuJM8gHWpHYpHABEBAAG0PURhbmllbCBIaXhvbiAo
	Q2hpZWYgVGVjaG5vbG9naXN0KSA8ZGFuLmhpeG9uQGNoYXR0ZXJwbHVnLmNvbT6J
	ATgEEwECACIFAlNQQGMCwwMGCwkIBwmCBhUIAgkKCwQWAgMBAh4BAheAAAoJECZr
	H2SjRyNlGbQH/Rx3meb6fwnYy3aHdE7EMkgr9zKZd9pZgHR4X8DFuhPzpjy2+sDP
	rIbLToqKqPMiLfFhR73W/y93Ajd/xqplYPlthDn/oInYouk0jegNllzs5Kh1drf4
	kMlOkxTFxww7tHMW8Z0lk02KGbtb8cpt8uBOTevWApcFqBa0drn/9ZEnlG9mbdA6
	f4gnwQ9GTYqTxuaaL87MtKi0kqeRH3/Ht9lBUYd0nVxvA/5LwTslS4jqI5Ims8dy
	osepVszRJP+KDogCsRxb4kjeTESIjxJOG9Wu+d1shHFL6SOVfyToUv3UctuTJB7v
	tlqspz82CWUs+9g4j0Q5HDWBhIgKxjQMMxu5AQ0EU1BAYwEIANf3arPgjsuKJCLC
	9wddRP5hcLHOIqEiRDHCMeHjBsfExVDA/Obpjr2oFPSh1DOqyfjWtXM3zCcrLQyN
	xpn8ar5zd86RYOe4F4DMvJ5KKA7MNOqCsn0wVex6VatnAklf4J/YLIMy0XHaIvCY
	EV3myCLLQvS39tQj7RC8CC5lCsz3k6oenEzE7bWgUkYWhJs4DpFnllm3evV7GX1d
	CJCV0RJNPLX/oaUR4w5OX25lON3KpPMZNHMH9ZRwkL+9ZiDLlI1UAd80wHBGSpQ+
	oKI5g9My3VYyPg2+IDwbYDhtcxJQHMkSSzzYkX7y5W2b1JiQAiYtNknrltbkohap
	TckMhbUAEQEAAYkBHwQYAQIACQUCU1BAYwIbDAAKCRAmax9ko0cjZYhfB/9knMaB
	OaDQ6Pq4teM1AZfcZNjMqRhk0gsuFzyFtoABmnDOHsx/jx0GMutreDYEPIBgblzr
	X4DigSDZAXMrxGbplqbBBpRBP6Z/28BJ2SFWL28/eys1inbi09qSYja74QxigJc6
	fFyll5Ufx4V51I68Bit3tsUC/PYyZKaeTTrBXYPpqyfBHb3sKnWeXlwk4Bf+Azi4
	nqkSzz7n/EllURI15b72Wn2Thi8svEQz0fcYJyXrigj8NKl9Wlso7Wodo33cZ1S6
	00IoFuL+BOk8HfT6cLSjVSDy4CHHbCG25MCo1xIGSKeDJJYrFyPK8Jx5hw9HvMHl
	Lotwx9uv6tWMu9YY
	=SBwH
	-----END PGP PUBLIC KEY BLOCK-----
	KEY
		
	  # Specify mode (:asymmetric, :symmetric or :both)
	  gpg.mode = :both # defaults to :asymmetric
		# Specify recipients from #keys (for :asymmetric encryption)
	  gpg.recipients = ['dan.hixon@email.com']
	  # Specify passphrase or passphrase_file (for :symmetric encryption)
	  #encryption.passphrase = ENV['ENCRYPTION_PASSPHRASE']
	  gpg.passphrase_file = '~/backup_passphrase'
	end

	Notifier::Mail.defaults do |mail|
	  mail.on_success           = true
	  mail.on_warning           = true
	  mail.on_failure           = true
	  mail.from                 = 'sender@email.com'
	  mail.to                   = 'receiver@email.com'
	  mail.address              = 'smtp.gmail.com'
	  mail.port                 = 587
	  mail.domain               = 'your.host.name'
	  mail.user_name            = 'sender@email.com'
	  mail.password             = 'my_password'
	  mail.authentication       = 'plain'
	  mail.encryption           = :starttls
	end

###schedule.rb
Note that I've changed the job template to import the container environment variables.

	set :output, "/root/cron.log"
	# Get the environment variables from /etc/container_environment.sh
	set :job_template, "bash -l -c 'source /etc/container_environment.sh ; :job'"

	every 1.day, :at => '5:09 am' do
	  command "/usr/local/bin/backup perform -t daily_backup"
	end

	every :hour do
		command "/usr/local/bin/backup perform -t hourly_backup"
	end
