<template>
	<div>
		<el-form label-width="80px" :model="form" :rules="rules" ref="appointForm">
			<el-row>
				<el-col :span="7" v-if="userRole !== 'role_3'">
					<el-form-item label="会议室" prop="room">
						<el-select v-model="form.room" placeholder="请选择会议室">
				      <el-option
				      	v-for="room in RoomList"
				      	:key="room.value"
				      	:label="room.label"
				      	:value="room.value"
				      >
				      </el-option>
				    </el-select>
					</el-form-item>
				</el-col>
				<el-col :span="7">
					<el-form-item label="预约日期" required>
						<el-date-picker
				      v-model="choosedDay"
				      type="date"
				      placeholder="选择日期"
				      @change="onChoosedDayChange"
				      :picker-options="pickerOptions">
				    </el-date-picker>
					</el-form-item>
				</el-col>
			</el-row>
			<el-row v-if="userRole !== 'role_3'">
				<el-col :span="7">
					<el-form-item label="开始时间" prop="startTime" required>
						<el-time-select
					    placeholder="开始时间"
					    v-model="form.startTime"
					    :picker-options="{
					      start: '09:00',
					      step: '00:30',
					      end: '18:00'
					    }">
				  	</el-time-select>
				  </el-form-item>
				</el-col>
			  <el-col :span="7">
			  	<el-form-item label="结束时间" prop="endTime" required>
				  	<el-time-select
					    placeholder="结束时间"
					    v-model="form.endTime"
					    :picker-options="{
					      start: '09:00',
					      step: '00:30',
					      end: '18:00',
					      minTime: form.startTime
					    }">
				  	</el-time-select>
			  	</el-form-item>
			  </el-col>
				<el-col :span="4">
			    <el-button type="primary" @click="onSubmit('appointForm')">预约会议室</el-button>
			  </el-col>
		  </el-row>
		</el-form>
		<div class="room-use">
			<div class="room-title">会议室使用情况</div>
			<room-using
			  ref="roomUsingChart"
			/>
		</div>
	</div>
</template>

<script>
import RoomUsing from '../chart/roomUsing.vue'
import { getRoomList, appointRoom } from '@/front/api'
import { dateFormat, getKeyByValue } from '@/front/utils'
import appointTime from '@/front/utils/appointTime.js'

export default {
  created () {
    let userInfo = localStorage.getItem('MEETING_INFO')
    if (userInfo) {
      userInfo = JSON.parse(userInfo)
      this.userRole = userInfo.userRole
    }
    this.getRoomListData()
  },
  data () {
    return {
      form: {
        room: '',
        startTime: '',
        endTime: ''
      },
      rules: {
        room: [
          { required: true, message: '请选择会议室' }
        ],
        startTime: [
          { required: true, message: '请选择开始时间', trigger: 'change' }
        ],
        endTime: [
          { required: true, message: '请选择结束时间', trigger: 'change' }
        ]
      },
      pickerOptions: {
        disabledDate (time) {
          return time.getTime() < Date.now() - 8.64e7
        }
      },
      choosedDay: dateFormat(new Date(), 'yyyy-MM-dd'),
      RoomList: [],
      userRole: 'role_3'
    }
  },
  methods: {
    onSubmit (formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          const appointData = {
            room: this.form.room,
            roomUser: 'YJF',
            usingDay: this.choosedDay,
            startTime: getKeyByValue(this.form.startTime, appointTime),
            endTime: getKeyByValue(this.form.endTime, appointTime)
          }
          // console.log('submit', appointData)
          if (this.checkRoomUse(appointData.startTime, appointData.endTime)) {
            appointRoom(appointData)
            .then(({ msg }) => {
              this.$message({
                message: msg,
                type: 'success',
                duration: 2000
              })
            })
            .then(() => {
              this.$refs.roomUsingChart.getRoomsData(this.choosedDay)
            })
            .catch(({ code, msg }) => {
              this.$message({
                message: msg,
                type: 'error',
                duration: 2000
              })
            })
          } else {
            this.$message({
              message: '预约时间有冲突哦～',
              type: 'error',
              duration: 1000
            })
          }
        } else {
          return false
        }
      })
    },
    onChoosedDayChange (choosedDay) {
      if (choosedDay) {
        this.choosedDay = choosedDay
        this.$refs.roomUsingChart.getRoomsData(choosedDay)
      }
    },
    getRoomListData () {
      getRoomList()
      .then(({ records }) => {
        this.RoomList = records.map(({ roomName }) => ({ label: roomName, value: roomName }))
      })
      .catch(({ code, msg }) => {
        this.$message({
          message: msg,
          type: 'error',
          duration: 2000
        })
      })
    },
    checkRoomUse (startTime, endTime) {
      const roomUseData = this.$refs.roomUsingChart.roomUsingData
      let isPass = true
      console.log(roomUseData, this.form.room)
      if (roomUseData.length !== 0) {
        const curRoomuseData = roomUseData.filter(item => item.room === this.form.room)
        if (curRoomuseData.length !== 0) {
          const [{ roomUse }] = curRoomuseData
          roomUse.forEach(oldRoom => {
            if (startTime >= oldRoom.usingTime[0] && startTime < oldRoom.usingTime[1]) {
              isPass = false
            }
            if (endTime > oldRoom.usingTime[0] && endTime <= oldRoom.usingTime[1]) {
              isPass = false
            }
            if (startTime <= oldRoom.usingTime[0] && endTime >= oldRoom.usingTime[1]) {
              isPass = false
            }
          })
        }
      }
      return isPass
    }
  },
  components: {
    RoomUsing
  }
}
</script>

<style>
.room-use {
	height: 100%;
	font-size: 14px;
}

.room-title {
	text-align: center;
}
</style>