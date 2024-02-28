# LRU-K

将传统 LRU 中加入 FIFO 列表，如果一个数据没有访问 K 次，那么它在 FIFO 列表中，当访问到 K 次后，加入 LRU 中。

FIFO 中使用传统的先进先出策略，在 LRU 中，按照当前次的时间戳和 K 次前的时间戳，选择差距最大的进行淘汰

## 为什么使用 LRU-K

为了解决缓存污染，防止热点数据被刷下去，同时防止一些只访问一两次的数据将热点数据替换下去

## 源码

```cpp
//
// Created by Anti on 2022/12/27.
//

#include "LRU_K.h"
LRUKReplacer::LRUKReplacer(size_t num_frames, size_t k) : replacer_size_(num_frames), k_(k) {
    max_size_=num_frames;
}

auto LRUKReplacer::Evict(frame_id_t *frame_id) -> bool {
    std::lock_guard<std::mutex> lock(latch_);
    /**
     * 如果没有可以驱逐元素
     */
    if(Size()==0)
    {
        return false;
    }
    /**
     * 首先尝试删除距离为无限大的缓存
     */
    for(auto it = new_frame_.rbegin();it!=new_frame_.rend();it++)
    {
        auto frame = *it;
        if(evictable_[frame])//如果可以被删除
        {
            recorded_cnt_[frame] = 0;
            new_locate_.erase(frame);
            new_frame_.remove(frame);
            *frame_id = frame;
            curr_size_--;
            hist[frame].clear();
            return true;
        }
    }
    /**
     * 再尝试删除已经访问过K次的缓存
     */
    for(auto it =cache_frame_.begin();it!=cache_frame_.end();it++)
    {
        auto frame = (*it).first;
        if(evictable_[frame])
        {
            recorded_cnt_[frame] = 0;
            cache_frame_.erase(it);
            cache_locate_.erase(frame);
            *frame_id = frame;
            curr_size_--;
            hist[frame].clear();
            return true;
        }
    }
    return false;
}

void LRUKReplacer::RecordAccess(frame_id_t frame_id)
{
    std::lock_guard<std::mutex> lock(latch_);

    if(frame_id>static_cast<frame_id_t>(replacer_size_))
    {
        throw std::exception();
    }
    current_timestamp_++;
    recorded_cnt_[frame_id]++;
    auto cnt = recorded_cnt_[frame_id];
    hist[frame_id].push_back(current_timestamp_);
    /**
     * 如果是新加入的记录
     */
    if(cnt==1)
    {
        if(curr_size_==max_size_)
        {
            frame_id_t frame;
            Evict(&frame);
        }
        evictable_[frame_id] = true;
        curr_size_++;
        new_frame_.push_front(frame_id);
        new_locate_[frame_id] = new_frame_.begin();
    }
    /**
     * 如果记录达到k次，则需要从新队列中加入到老队列中
     */
    if(cnt==k_)
    {
        new_frame_.erase(new_locate_[frame_id]);//从新队列中删除
        new_locate_.erase(frame_id);

        auto kth_time = hist[frame_id].front();//获取当前页面的倒数第k次出现的时间
        k_time new_cache(frame_id,kth_time);
        auto it = std::upper_bound(cache_frame_.begin(),cache_frame_.end(),new_cache,CmpTimestamp);//找到该插入的位置
        it = cache_frame_.insert(it,new_cache);
        cache_locate_[frame_id] = it;
        return;
    }
    /**
     * 如果记录在k次以上，需要将该frame放到指定的位置
     */
    if(cnt>k_)
    {
        hist[frame_id].erase(hist[frame_id].begin());
        cache_frame_.erase(cache_locate_[frame_id]);//去除原来的位置
        auto kth_time = hist[frame_id].front();//获取当前页面的倒数第k次出现的时间
        k_time new_cache(frame_id,kth_time);

        auto it = std::upper_bound(cache_frame_.begin(),cache_frame_.end(),new_cache, CmpTimestamp);//找到该插入的位置
        it = cache_frame_.insert(it,new_cache);
        cache_locate_[frame_id] = it;
        return;
    }
    /**
     * 如果cnt<k_，是不需要做更新动作的
     */
}

void LRUKReplacer::SetEvictable(frame_id_t frame_id, bool set_evictable)

{
    std::lock_guard<std::mutex> lock(latch_);
    if(recorded_cnt_[frame_id]==0)
    {
        return ;
    }
    auto status = evictable_[frame_id];
    evictable_[frame_id] = set_evictable;
    if(status&&!set_evictable)
    {
        --max_size_;
        --curr_size_;
    }
    if(!status&&set_evictable)
    {
        ++max_size_;
        ++curr_size_;
    }
}

void LRUKReplacer::Remove(frame_id_t frame_id) {
    std::lock_guard<std::mutex> lock(latch_);
    if (frame_id > static_cast<frame_id_t>(replacer_size_)) {
        throw std::exception();
    }
    auto cnt = recorded_cnt_[frame_id];
    if (cnt == 0)
    {
        return ;
    }
    if(!evictable_[frame_id])
    {
        throw std::exception();
    }
    if(cnt<k_)
    {
        new_frame_.erase(new_locate_[frame_id]);
        new_locate_.erase(frame_id);
        recorded_cnt_[frame_id] = 0;
        hist[frame_id].clear();
        curr_size_--;
    }
    else
    {
        cache_frame_.erase(cache_locate_[frame_id]);
        cache_locate_.erase(frame_id);
        recorded_cnt_[frame_id] = 0;
        hist[frame_id].clear();
        curr_size_--;
    }
}

auto LRUKReplacer::Size() -> size_t { return curr_size_; }
auto LRUKReplacer::CmpTimestamp(const LRUKReplacer:: k_time &f1,const LRUKReplacer:: k_time &f2) -> bool {
    return f1.second<f2.second;
}
```

