task default: %w[setup]

require "json"

ORGANIZATION, REPOSITORY = ENV["WMID_REPOSITORY"].split("/")

TMP = "/tmp"
CWD = "#{TMP}/#{ORGANIZATION}"
RWD = "#{TMP}/#{ORGANIZATION}/#{REPOSITORY}"

LABELS = ENV["GITHUB_LABELS"].split(",").map { |l| l.strip }

ANNOTATIONS = "annotations"

task setup: %w[aliases repo sync] do
end

task :aliases do
  sh %{ gh alias set labels 'api repos/{owner}/{repo}/labels' }
  sh %{ gh alias set unassigned 'issue list --search \"no:assignee\"' }
  sh %{ gh alias set unlabeled 'issue list --search \"is:issue is:open #{LABELS.map { |l| "-label:#{l}" }.join(" ")}\"' }
  sh %{ gh alias set --shell unleveled 'grep -LE \"^level: \" issues/*' }
  sh %{ gh alias set --shell Level 'gsed -i "1i\level: $2" issues/$1' }
  sh %{ gh alias set --shell View 'echo "$(grep -s "level: " issues/$1)\n$(gh issue view $1)" | cat -' }
  sh %{ gh alias set curate "for u in $(gh unattended --json number --jq ".[].number"); do }
end

desc "Pull new issues."
task :new do

  issues =
    %x{ gh issue list --limit 9999 --json number --jq ".[].number" }
    .split("\n")

  issues.each do |issue|
    sh %{ mkdir #{ANNOTATIONS}/#{issue} } unless File.exist?("#{ANNOTATIONS}/#{issue}")
  end

end

desc "Update the issues."
task update: [:new] do

  issues =
    %x{ ls annotations/ }
    .split("\n")

  issues.each do |issue|
    sh "gh issue view #{issue} > #{ANNOTATIONS}/#{issue}/#{issue}"
  end

end


task repo: CWD do
  chdir CWD
  sh "git clone --depth 1 https://github.com/#{ENV["WMID_REPOSITORY"]}.git"

  chdir RWD
  sh "git remote add coordinator git@github.com:#{ENV["WMID_COORDINATOR"]}/#{REPOSITORY}.git"
  sh "git fetch --depth 1 coordinator #{ENV["WMID_COORDINATOR"]}"

  sh "git checkout coordinator/#{ENV["WMID_COORDINATOR"]}"

end

file CWD do
  chdir TMP
  mkdir CWD
end

task :sync do
  chdir RWD

  open = %x{ gh issue --repo #{ENV["WMID_REPOSITORY"]} list --limit 999 --json number --jq ".[].number" }

  open.split("\n").each do |number|
    sh %{ gh View #{number} | tee "issues/#{number}.cache" }
    sh %{ mv issues/#{number}.cache issues/#{number} }
  end

end


task :clean do
  puts "rm -r -f #{CWD}"
end


namespace :auto do

  multitask update: %w[assign label review] do
    chdir RWD
  end

  task :unattended do
    sh %{ for u in $(gh unattended --json number --jq ".[].number"); do curl https://maker.ifttt.com/trigger/notify/with/key/$IFTTT_KEY -d value1="$(gh issue view $u --json number --jq '.number')" -d value2="$(gh issue view $u --json title --jq '.title')" -d value3="$(gh issue view $u --json url --jq '.url')"; done; }
  end

  task :assign do
    chdir RWD
    sh %{
      for issue in $(gh issue list --search "no:assignee" --json number --jq ".[].number");
        do gh wmid assignee $issue #{ENV["WMID_COORDINATOR"]};
      done;
    }
  end

  task :label do
    chdir RWD
    sh %{
    for unattended in $(gh unattended --json number --jq ".[].number"); do echo $unattended; gh wmid label $unattended "$(gh wmid assignment assist $unattended)"; done
    }
  end

  task :review do
    chdir RWD
    sh %{ for pr in $(gh pr list --search "-assignee:#{ENV['WMID_COORDINATOR']}" --json number --jq ".[].number"); do gh wmid assignee $pr #{ENV['WMID_COORDINATOR']}; done }
    sh %{gh pr list}
  end

  task :participants do
    chdir RWD

    json  = %x{gh issue list --search "-label:writer-input-required created:>=2022-01-01" --json number,comments,url --limit 20}

    JSON.parse(json).each do |issue|
      issue["comments"].each do |comment|
        if comment["body"].match(/(@[0-9a-z_]+)/)
          mentioned = $1

          title = "##{issue["number"]}"
          body  = "@#{comment["author"]["login"]} has mentioned #{mentioned}"

          puts "#{title} : #{body}"

          sh %{
          curl https://maker.ifttt.com/trigger/notify/with/key/#{ENV["IFTTT_KEY"]} -d value1="#{title}" -d value2="#{body}" -d value3="#{issue["url"]}"
          }

        end
      end
    end

  end

end

namespace :openai do

  labels = ENV["GITHUB_LABELS"].split(",").map { |l| l.strip }
  label_objects = labels.map { |l| "#{l}.json" }

  task submit: "training.json" do
    sh %{
      curl https://api.openai.com/v1/files \
      -H "Authorization: Bearer $OPENAI_API_KEY" \
      -F purpose="classifications" \
      -F file="@training.json" \

      | grep "file-" \
      | cut -f2 -d: | tr -d "," \
      > training_id.txt
    }
  end

  file "training.json": label_objects do
    sh %{ cat #{label_objects.join(" ")} > training.json }
  end

  labels.each do |label|
    file "#{label}.json" do

      sh %{ gh issue list \
            --search "org:MicrosoftDocs label:#{label}" \
            --json body --limit 125 \
            --jq ".[] | {\\"text\\": .body, \\"label\\": \\"#{label}\\"}" \
            | shuf -n 25 \
            > #{label}.json
      }

    end
  end

  task :env do
    puts ENV["OPENAI_REPOSITORY_FILE"]
  end

  task :clean do
    labels.each do |label|
      rm "#{label}.json"
    end
  end

end
