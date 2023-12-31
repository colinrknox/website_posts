# Creating My Site

## Starting Out
Throughout the past year, I've noticed many job applications ask for your website. I didn't think much of it, but then I started to think about the different ways you could have your "own" blog or personal website. There's sort of a spectrum of options. On one end, you have websites like Medium that just require you to sign up for an account and all you have to bring are the posts. On the other end, you have to run all of your hardware out of your home and manage the machine yourself. I decided on somewhere in the middle by using AWS services and a domain registrar and it's nameservers. For the website itself, I decided on a small Go back end. It's not necessary but I just wanted to get more familiar with Go. I'm also using Tailwind for the styling.  

## Technology Choices
My choice of tech stack ultimately came down to a combination of their popularity, my interest in them, and whether or not I have used them before. I chose React because I have never really used React before and I wanted to see how much different from Angular it could be. I decided to add in Tailwind mostly to avoid having to work with Bootstrap and build everything from the ground up with CSS. The website is simply a splash page with links to blog posts. The backend is a bit more interesting and came about because I wanted to do something with Go. Instead of just writing my posts directly in the React files I wanted a more general form of them. I decided I would write the posts in markdown directly to a GitHub repository and have it dynamically updated on my website. I figured I could just create some sort of batch job on the EC2 to periodically pull from the repository and copy the contents into a folder that I would serve with Go. That's how I learned that Unix has a system-level CRON job executor. All you have to do is ```crontab -e``` and add a CRON expression and a shell script and the operating system will take care of the rest. Go has standard library support for simple file servers, however, I wanted the ability to have REST endpoints so I ended up using Gin.

## Implementation
There are four main parts to this website. The front end is built with React and Tailwind and it is hosted currently on Cloudfront through Amplify and a Route 53 hosted zone. The web server is written in Go and lives as a systemctl daemon on an EC2 instance. All of the code and blog posts are hosted on Github. The final component is the Unix CRON job that runs according to a schedule to pull new blog posts from the post repository and process and stage the files for the Go web server. I chose to do it this way so that I could write the posts in a format that is as close to universal as it can get, markdown. This setup also allows me to update the content on the website without having to push new builds of the React component. I push the requisite files for a new post to the post repository. Then, the EC2 instance will automatically pull new commits and process and stage the new content for the Go server.  

Here's a breakdown of my batch job script:  

    REPO_DIR="/home/ubuntu/website_posts"
    TARGET_DIR="/home/ubuntu/public"
    LOG_FILE="/home/ubuntu/logs/batch.log"

    touch "$LOG_FILE"                                                 (1)
    cd "$REPO_DIR"
    git pull

    find "$REPO_DIR" -name '*.md' | while read md_file; do            (2)
	    json_file="$TARGET_DIR/$(basename "${md_file%.*}").json"
	    img_file="$(basename "${md_file%.*}").png"
	    title=$(basename "${md_file%.*}" | sed 's/_/ /g')
	    date=$(date --iso-8601)

	    if [[ -f "$json_file" ]]; then
  		  echo "$(date '+%Y-%m-%d %H:%M:%S') - Updating $json_file contents" >> "$LOG_FILE"
		    jq --arg contents "$(<"$md_file")" '.contents = $contents' "$json_file" > tmp.json && mv tmp.json "$json_file"
	    else
		    echo "$(date '+%Y-%m-%d %H:%M:%S') - Creating new JSON file "$json_file"" >> "$LOG_FILE"
		    jq -n \
			    --arg title "$title" \
			    --arg date "$date" \
			    --arg img_url "$img_file" \
			    --arg contents "$(<"$md_file")" \
			    '{title: $title, date: $date, imgUrl: $img_url, contents: $contents}' > "$json_file"
	    fi
    done

    echo "$(date '+%Y-%m-%d %H:%M:%S') - Copying png files to target directory" >> "$LOG_FILE"        (3)
    find "$REPO_DIR" -name '*.png' -exec cp {} "$TARGET_DIR" \;
1. Ensure the log file exists and create it if it doesn't. Get the latest commits for the repository.
2. Find all the markdown files in the directory and loop through each. If the JSON file for a markdown file doesn't exist, then create it, if it does exist, then only update the contents and leave the other metadata in the existing JSON alone.
3. Copy all the PNG image files to the directory that the Go backend serves files from.  

The rest of the codebase can be found on my Github which has a link at the bottom of the page.

## The importance of building things
I think reading and educating yourself on different topics and technologies is extremely valuable. But an important aspect of software engineering is just working on and building things. It's amazing the amount of information you're able to remember. It also helps me build a better mental model of systems and how they work. It's something that you have to approach from the mindset of getting better just a little bit every day. You can always improve as an engineer because the sky is the limit.

## Looking Forward
My first goal is to get some more technical posts on this site. After that, I think my next idea was to integrate my Lox interpreter into the site using WASM. At this point, I can go multiple ways with it. That does not include the potential ideas that may arise over time.
