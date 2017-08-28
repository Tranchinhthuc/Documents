## **Deploy Rails app lên Digital Ocean với Nginx, unicorn, capistrano.** ##

-----------------------------------------------------
**Deploy với Nginx + unicorn.**

> Ta sẽ không deploy trực tiếp rails app trên tài khoản root của server. Thay vì vậy, tạo 1 user và deploy trên user này, trong bài này, sẽ đặt tên user là deploy. Tham khảo cách tạo 1 user trên ubuntu: https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps

Cài đặt môi trường cần thiết để chạy được 1 app rails, giống như khi cài đặt trên máy local. Trong bài hướng dẫn này sử dụng rbenv để quản lý ruby version. Tham khảo https://gorails.com/setup/ubuntu/14.04

Clone 1 dự án về(nên dùng ssh key để clone).

    git clone remote_git_path

Sửa config/database.yml, chạy bundle, migrate

    RAILS_ENV=production rake db:create db:migrate
Chạy dự án bằng lệnh:
		

    $ rails server -b 0.0.0.0
Vào http://SERVER_PIBLIC_IP:3000 để xác nhận kết qủa.

----------
**Cài đặt Unicorn:**
Thêm gem unicorn vào Gemfile.

    # Gemfile
    gem 'unicorn'
Chạy bundle để cài đặt gem

    bundle install
Có thể start server bằng lệnh:

    $ bundle exec unicorn -c config/unicorn.rb -E production

Cổng mặc định của unicorn là 8080. Vào http://SERVER_PUBLIC_IP:8080 để xem kết qủa. Có thể thêm option -D(bundle exec unicorn -c config/unicorn.rb -E production -D) để chạy server ở deamon mode(chế độ chạy ngầm).

 Tạo 1 file config/unicorn.rb để cấu hình unicorn, cấu hình này sẽ được áp dụng mỗi khi start server.
 

	    root = "/home/deploy/appname/current"
		working_directory root
		#pid của unicorn khi chạy
		pid "#{root}/tmp/pids/unicorn.pid"
		#log
		stderr_path "#{root}/log/unicorn.error.log"
		stdout_path "#{root}/log/unicorn.access.log"
		
		listen "/home/deploy/appname/shared/sockets/unicorn.sock"
		worker_processes 2
		timeout 30
		
		
		before_fork do |server, worker|
		  Signal.trap 'TERM' do
		    puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
		    Process.kill 'QUIT', Process.pid
		  end
		
		  defined?(ActiveRecord::Base) and
		    ActiveRecord::Base.connection.disconnect!
		end
		
		after_fork do |server, worker|
		  Signal.trap 'TERM' do
		    puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
		  end
		
		  defined?(ActiveRecord::Base) and
		    ActiveRecord::Base.establish_connection
		end
		
		# Force the bundler gemfile environment variable to
		# reference the capistrano "current" symlink
		before_exec do |_|
		  ENV['BUNDLE_GEMFILE'] = File.join(root, 'Gemfile')
		end

Tại thư mục của rails app, chạy lệnh để tạo các folder sẽ sử dụng sau đó:

    $ mkdir -p shared/pids shared/sockets shared/log
 
**Tạo 1 file script để có thể start/stop unicorn.**

    $ sudo vi /etc/init.d/unicorn_appname
 
**Nội dung**
    
    #!/bin/sh
    ### BEGIN INIT INFO
    # Provides:          unicorn
    # Required-Start:    $all
    # Required-Stop:     $all
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: starts the unicorn app server
    # Description:       starts unicorn using start-stop-daemon
    ### END INIT INFO
    
    set -e
    
    USAGE="Usage: $0 <start|stop|restart|upgrade|rotate|force-stop>"
    
    # app settings
    USER="deploy"
    APP_NAME="appname"
    APP_ROOT="/home/$USER/$APP_NAME"
    ENV="production"
    
    # environment settings
    PATH="/home/$USER/.rbenv/shims:/home/$USER/.rbenv/bin:$PATH"
    CMD="cd $APP_ROOT && bundle exec unicorn -c config/unicorn.rb -E $ENV -D"
    PID="$APP_ROOT/shared/pids/unicorn.pid"
    OLD_PID="$PID.oldbin"
    
    # make sure the app exists
    cd $APP_ROOT || exit 1
    
    sig () {
      test -s "$PID" && kill -$1 `cat $PID`
    }
    
    oldsig () {
      test -s $OLD_PID && kill -$1 `cat $OLD_PID`
    }
    
    case $1 in
      start)
        sig 0 && echo >&2 "Already running" && exit 0
        echo "Starting $APP_NAME"
        su - $USER -c "$CMD"
        ;;
      stop)
        echo "Stopping $APP_NAME"
        sig QUIT && exit 0
        echo >&2 "Not running"
        ;;
      force-stop)
        echo "Force stopping $APP_NAME"
        sig TERM && exit 0
        echo >&2 "Not running"
        ;;
      restart|reload|upgrade)
        sig USR2 && echo "reloaded $APP_NAME" && exit 0
        echo >&2 "Couldn't reload, starting '$CMD' instead"
        $CMD
        ;;
      rotate)
        sig USR1 && echo rotated logs OK && exit 0
        echo >&2 "Couldn't rotate logs" && exit 1
        ;;
      *)
        echo >&2 $USAGE
        exit 1
        ;;
    esac
   **Cập nhập permissions cho phép chạy file này**
   

    sudo chmod 755 /etc/init.d/unicorn_appname
    sudo update-rc.d unicorn_appname defaults
   Chạy lệnh sau để start server unicorn
   

    $ sudo service unicorn_appname start
   Đến đây thì rails app đang chạy ở môi trường production bằng server unicorn. Unicorn là server trung gian, ở giữa tầng web service(nginx, apache...) và tầng ứng dụng(rails app). Vì vậy, để truy cập vào app từ bên ngoài, ta cần kết nối web service(trong bài này dùng nginx) với unicorn. Ta cài đặt nginx.
   

    sudo apt-get install nginx
Cấu hình nginx

    sudo vim /etc/nginx/sites-available/cap_test
Nội dung như sau

      upstream unicorn {
		  server unix:/home/deploy/cap_test/current/shared/sockets/unicorn.sock fail_timeout=0;
		}
		
		server {
		  listen 80 default deferred;
		  server_name _;
		  root /home/deploy/cap_test/current/public;
		
		  location ^~ /assets/ {
		    gzip_static on;
		    expires max;
		    add_header Cache-Control public;
		  }
		
		  try_files $uri/index.html $uri @unicorn;
		  location @unicorn {
		    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		    proxy_set_header Host $http_host;
		    proxy_redirect off;
		    proxy_pass http://unicorn;
		  }
		
		  error_page 500 502 503 504 /500.html;
		  client_max_body_size 4G;
		  keepalive_timeout 10;
		}
Chú ý đường dẫn rails_app_path/shared/sockets/unicorn.sock là đường dẫn để kết nối với unicorn thông qua file .sock. Và đường dẫn rails_app_path/public là đường dẫn tới thư mục public của rails app.
Giờ ta enable file này:

    $ sudo ln -s /etc/nginx/sites-available/cap_test /etc/nginx/sites-enabled/cap_test
    $ sudo rm –rf /etc/nginx/sites-enabled/default
Giờ ta kiểm tra lại bằng:

    $ sudo /etc/init.d/nginx restart
    $ sudo /etc/init.d/unicorn start
Vào http://SERVER_PUBLIC_IP để kiểm tra.

Như vậy là ta vừa deploy thủ công 1 rails app với nginx và unicorn. Giờ ta cấu hình capistrano để việc deploy được tự động. Capistrano giúp ta chạy tự động 1 số task đặc trưng của 1 rails app như migrate, compile asset, ... hoặc các task tự định nghĩa. Thêm các gem sau vào Gemfile, sau đó chạy lệnh bundle:

      gem 'capistrano',         require: false
	  gem 'capistrano-rvm',     require: false
	  gem 'capistrano-rails',   require: false
	  gem 'capistrano-bundler', require: false
	  gem 'capistrano3-unicorn',   require: false
Khởi tạo file cấu hình bằng lệnh:

    $ bundle exec cap install
Câu lệnh trên tạo ra các file theo cấu trúc:

    ├── Capfile
	├── config
	│   ├── deploy
	│   │   ├── production.rb
	│   │   └── staging.rb
	│   └── deploy.rb
	└── lib
	    └── capistrano
	            └── tasks
Nội dung Capfile:

    require 'capistrano/setup'
	require 'capistrano/deploy'
	require 'capistrano/rvm'
	require 'capistrano/bundler'
	require 'capistrano/rails/assets'
	require 'capistrano/rails/migrations'
	require 'capistrano3/unicorn'
	# Load custom tasks from `lib/capistrano/tasks' if you have any defined
	# Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
Nội dung config/deploy.rb

    # config valid only for current version of Capistrano
	lock "3.9.0"
	
	set :application, "appname"
	set :repo_url, "link_to_git_repo"
	
	set :branch, ENV['BRANCH'] || "master"
	
	set :use_sudo, true
	set :bundle_binstubs, nil
	# set :linked_files, fetch(:linked_files, []).push('config/database.yml')
	set :linked_dirs, fetch(:linked_dirs, []).push('log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system')
	
	after 'deploy:publishing', 'deploy:restart'
	
	namespace :deploy do
	  task :restart do
	    invoke 'unicorn:reload'
	  end
	end
Nội dung config/deploy/[ENV].rb

    set :port, 22
	set :user, 'deploy'
	set :deploy_via, :remote_cache
	set :use_sudo, false
	
	server 'SERVER_PUBLIC_IP',
	  roles: [:web, :app, :db],
	  port: fetch(:port),
	  user: fetch(:user),
	  primary: true
	
	set :deploy_to, "/home/deploy/#{fetch(:application)}"
	
	set :ssh_options, {
	  forward_agent: true,
	  auth_methods: %w(publickey),
	  user: 'deploy',
	}
	
	set :rails_env, :production
	set :conditionally_migrate, true

Chạy lệnh sau để deploy:

    $ cap production deploy
Có thể rollback lại 1 version trước bằng lệnh:

    $ cap production deploy:rollback ROLLBACK_RELEASE=20171227172059
20171227172059 nằm trong thư mục releases/
